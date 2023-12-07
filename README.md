Bu app bazi endpoint'ler iceriyor:

/health
/togglesick     ==> Bir pod'u bozuyor
/hiccup?time=5  ==> Pod'u 5 saniyeligine bozuyor (ki bu sayede readiness probe durumu gorsun ve service ep'lerden cikarsin. Bu test esnasinda sureyi uzatmak faydali olabilir)

*** Once app'i deploy ediyoruz:

$ oc apply -f long-load-deploy.yaml

$ oc exec deploy/long-load -- curl -s localhost:3000/health
app is still starting

*** Aslinda app henuz 'ready' degil (START_DELAY ==> 30000) ancak henuz bir startupProbe eklemedigimiz icin Kubernetes'in durumdan haberi yok:

$ oc get pods
NAME                         READY   STATUS    RESTARTS   AGE
long-load-8564d998cc-579nx   1/1     Running   0          30s
long-load-8564d998cc-ttqpg   1/1     Running   0          30s
long-load-8564d998cc-wjtfw   1/1     Running   0          30s

*** long-load-deploy.yaml'a startupProbe ekliyoruz (ve pod'larin hizli Ready olmasi icin scale down/up yapiyoruz):

$ oc set probe deploy/long-load --startup  --failure-threshold 30 --period-seconds 3 --get-url http://:3000/health

$ oc scale deploy/long-load --replicas 0 ; oc scale deploy/long-load --replicas 3

*** 30 sn icinde baktigimizda pod'larin hazir olmadigini goruyoruz:

$ oc get pods
NAME                         READY   STATUS    RESTARTS   AGE
long-load-785b5b4fc8-7x5ln   0/1     Running   0          27s
long-load-785b5b4fc8-f7pdk   0/1     Running   0          27s
long-load-785b5b4fc8-r2nqj   0/1     Running   0          27s

*** Bir terminalde load-test.sh basliatiyoruz:

$ ./load-test.sh
app is still starting
app is still starting
app is still starting
...output omitted...
Ok
Ok
Ok

*** Diger terminalde /togglesick ile pod'lardan birini 'unhealthy' yapiyoruz:

$ oc exec deploy/long-load -- curl -s localhost:3000/togglesick

Ara sira unhealthy pod'dan da cevap geliyor ancak livenessProbe henuz eklenmedigi icin bir duzetlme olmuyor:

app is unhealthy
Ok
Ok
Ok
app is unhealthy
Ok
app is unhealthy
Ok

*** Liveness probe ekliyoruz:

$ oc set probe deploy/long-load --liveness --get-url=http://:3000/health --period-seconds=3 --failure-threshold=15

*** Pod'larin hizli update olmasi icin (aksi taktirde her birinin teker teker ready olmasi uzun 30'ar saniye surdugu icin uzun surer):

$ oc scale deploy/long-load --replicas 0 ; oc scale deploy/long-load --replicas 3

*** Dikkat! POD'lar 'ready' olduktan sonra yine 'togglesick' ile pod'lardan birini 'unhealthy' yapiyoruz. Bu defa bir sure sonra 'unhealthy' pod'un bu defa restart edildigini goruyoruz:

$ oc get pods
NAME                        READY   STATUS    RESTARTS       AGE
long-load-fbb7468d9-8xm8j   1/1     Running   0              9m42s
long-load-fbb7468d9-k66dm   1/1     Running   0              8m38s
long-load-fbb7468d9-ncxkh   0/1     Running   1 (11s ago)    10m

*** Son olarak 'readiness' probe ekliyoruz ve scale down/up yapiyoruz:

$ oc set probe deploy/long-load --readiness --failure-threshold 1 --period-seconds 3 --get-url http://:3000/health

$ oc scale deploy/long-load --replicas 0 ; oc scale deploy/long-load --replicas 3

*** Dikkat! POD'lar 'ready' olduktan sonra bu defa 'hiccup?time=10' ile kisa sureligine pod'u bozuyoruz. Bu durumda pod, "service endpoint"ten cikariliyor kisa bir sureligine (bu ornekte 10 saniyeligine (hiccup suredi) veya liveness probe restart edene kadar).

Diger bir terminal'den 'watch oc get ep' ile takip ederken:

$ oc exec deploy/long-load -- curl -s localhost:3000/hiccup?time=10


