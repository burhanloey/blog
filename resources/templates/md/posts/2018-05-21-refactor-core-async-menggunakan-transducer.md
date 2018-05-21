{:title "Refactor code core.async menggunakan Transducer"
 :layout :post
 :tags  ["concurrency" "clojure" "lisp"]}
 
Dulu saya hanya menulis code Clojure secara *synchronous*, dan hanya pindahkan
code ke *thread* lain bila perlu. Baru-baru ini saya jenguk-jenguk code untuk
[Ring](https://github.com/ring-clojure/ring), dan saya mula perasan setiap *middleware* mengambil dua jenis function, satu
function 1-*arity* yang biasa, satu lagi function 3-*arity*. Saya pun tertanya-tanya
apa guna function 3-*arity* ni.

Lepas check, rupa-rupanya function 3-*arity* itu adalah untuk *async handler*.
Server Clojure yang lain memang dah boleh handle *async request*, contohnya
[Aleph](https://github.com/ztellman/aleph) dan
[http-kit](https://github.com/http-kit/http-kit), tapi saya tak suka guna sebab
tak ikut standard Ring. Server default untuk Ring ialah Jetty. Versi Jetty yang
latest dah ikut spec Servlet 3.1, jadi dah boleh handle *async request*.

Jadi, untuk enable function 3-*arity*, kalau menggunakan [lein-ring](https://github.com/weavejester/lein-ring), cuma perlu set
option `:async?` ke `true`. Kemudian tukar handler dari,

```clojure
(defn hello-handler [req]
  {:status 200
   :headers {}
   :body "Hello"})
```

ke

```clojure
(defn hello-handler [req respond raise]
  (respond {:status 200
            :headers {}
            :body "Hello"}))
```

Maksudnya kalau kita panggil function `respond` dari mana-mana thread pun, dia
akan terus bagi response. Tak perlu guna blocking call dah.

Maka saya pun cubalah menulis handler menggunakan
[core.async](https://github.com/clojure/core.async). Konsep `core.async` sama
sahaja dengan konsep concurrency dalam programming language Go.

Ini cubaan pertama:

```clojure
(defn register [{:keys [params]} respond _]
  (let [form (select-keys params [:username :email :password])
        validated (chan)
        user (chan)
        new-user (chan)
        hashed (chan)
        result (chan)]
    (go (>! validated (v/validate-registration form)))
    (go (let [[errors data] (<! validated)]
          (if errors
            (>! result (-> (redirect "/daftar")
                           (flash {:errors errors
                                   :data data}))) ; assoc previous data
            (>! user data))))
    (go (let [u           (<! user)
              by-username (async/thread (find-user-by-username (:username u)))
              by-email    (async/thread (find-user-by-email (:email u)))]
          (if (or (<! by-username) (<! by-email))
            (>! result (-> (redirect "/daftar")
                           (flash {:message "Username/email sudah diambil. Sila daftar menggunakan username/email yang lain."})))
            (>! new-user u))))
    (go (>! hashed (update (<! new-user) :password hashers/derive)))
    (go (<! hashed)
        (>! result (-> (redirect "/login")
                       (flash "Anda sudah berjaya mendaftar. Sila log masuk ."))))
    (go (let [[res _] (alts! [result (timeout 10000)])]
          (respond res)))))
```

Ambik kau. Buruk nak mampus. Jalan memang jalan code-nya, logic semua ok, tapi
dah takde rupa functional programming.

Jadi, saya tengok balik
[documentation](https://clojuredocs.org/clojure.core.async/chan) untuk `chan`,
saya perasan ada parameter `xform`. Eh?!

Saya tahu guna core.async, saya tahu guna transducer, tapi tak tahu pulak boleh
guna core.async dengan transducer sama-sama.

Selepas refactor, code jadi begini:

```clojure
(def user-existed-msg "Username/email sudah diambil. Sila daftar menggunakan username/email yang lain.")
(def success-msg "Anda sudah berjaya mendaftar. Sila log masuk .")

(defn- make-flash-msg [[errors data]]
  (if errors
    [{:errors errors :data data} data]
    [nil data]))

(defn- hash-user-password [[errors user :as all]]
  (if errors
    all
    [errors (update user :password hashers/derive)]))

(defn- make-response [[errors _]]
  (if errors
    (-> (redirect "/daftar")
        (flash errors))
    (-> (redirect "/login")
        (flash success-msg))))

(def validate-xform
  (comp
   (map v/validate-registration)
   (map make-flash-msg)))

(def register-xform
  (comp
   (map hash-user-password)
   ;; TODO: Add one more function here in the middle to persist user
   (map make-response)))

(defn register [{:keys [params]} respond _]
  (let [validated (chan 1 validate-xform)
        result (chan 1 register-xform)]
    (go (>! validated (select-keys params [:username :email :password])))
    (go (let [[errors user :as all] (<! validated)]
          (cond
            ;; If already has errors, skip checking for existing user.
            errors (>! result all)
            ;; Check for existing user with same username or email.
            (or (<! (async/thread (find-user-by-username (:username user))))
                (<! (async/thread (find-user-by-email (:email user)))))
            (>! result [{:message user-existed-msg} user])
            ;; Return same data if there is no problem.
            :else (>! result all))))
    (go (let [[res _] (alts! [result (timeout 10000)])]
          (respond res)))))
```

Sekarang barulah pendek-pendek function tersebut.

Beza dengan yang asal ialah yang asal saya terus bagi result sebaik sahaja ada
error. Orang panggil *short circuit*. Masalah dengan cara tersebut ialah cara
tersebut merupakan *side-effect*. *Side-effect* sangat tidak digalakkan dalam
functional programming.

Jadi, saya tukar kepada menggunakan konsep monad. Saya jadikan function-function
untuk `xform` supaya return sebuah vector di mana item pertama ialah error dan
item kedua ialah data sebenar yang perlu return. Error tersebut akan
diselesaikan di pengakhiran iaitu dalam function `make-response`.

Gambar ini mungkin lebih jelas:

![Gambar menunjukkan konsep monad](/images/monad.png)

Masalah yang tinggal sekarang ialah panggilan ke `async/thread` untuk
`find-user-by-username` dan `find-user-by-email`. Masalahnya sebab saya tidak
boleh panggil `<!` di luar block `go`. Kalau boleh, bolehlah saya letak bahagian
tersebut sebagai xform.

Jadi, tengoklah dulu.

**Edit**: Ini code selepas saya guna
[eduction](https://clojuredocs.org/clojure.core/eduction). Taktahu lah sama ada
ini dikira best practice. Function `register` jadi lagi pendek, kira ok lah tu.

```clojure
(def user-existed-msg "Username/email sudah diambil. Sila daftar menggunakan username/email yang lain.")
(def success-msg "Anda sudah berjaya mendaftar. Sila log masuk .")

(defn- make-flash-msg [[errors data]]
  (if errors
    [{:errors errors :data data} data]
    [nil data]))

(def validate-xform
  (comp
   (map v/validate-registration)
   (map make-flash-msg)))

(defn- validate [form]
  (let [validated (chan 1 validate-xform)]
    (put! validated form)
    validated))

(defn- find-existing-user [validated]
  (go (let [[errors user :as all] (<! validated)
            already-existed (chan)]
        (if errors
          (put! already-existed false)
          (let [by-username (async/thread (find-user-by-username (:username user)))
                by-email (async/thread (find-user-by-email (:email user)))]
            (put! already-existed (boolean (or (<! by-username) (<! by-email))))))
        (conj all already-existed))))

(defn- check-existing-user [user-check]
  (go (let [[errors user already-existed :as all] (<! user-check)]
        (cond
          errors all
          (<! already-existed) [{:message user-existed-msg} user]
          :else all))))          

(defn- hash-user-password [new-user]
  (go (let [[errors user :as all] (<! new-user)]
        (if errors
          all
          [errors (async/thread (update user :password hashers/derive))]))))

(defn- persist-user [new-user]
  (go (let [[errors hashed-user :as all] (<! new-user)]
        (if errors
          all
          (do
            (println "Registered: " (<! hashed-user))
            all)))))

(defn- make-response [registered]
  (go (let [[errors _] (<! registered)]
        (if errors
          (-> (redirect "/daftar")
              (flash errors))
          (-> (redirect "/login")
              (flash success-msg))))))

(def register-xform
  (comp
   (map validate)
   (map find-existing-user)
   (map check-existing-user)
   (map hash-user-password)
   (map persist-user)
   (map make-response)))

(defn register [{:keys [params]} respond _]
  (let [form (select-keys params [:username :email :password])
        [response] (eduction register-xform [form])]
    (go (let [[res _] (alts! [response (timeout 10000)])]
          (respond res)))))
```

**Edit 2**: Saya cuba *load testing* server kemudian membuat request untuk
handler tersebut. Server selamba je bagi response. Menarik.
