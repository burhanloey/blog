{:title "Pembetulan post lepas mengenai core.async"
 :layout :post
 :tags  ["concurrency" "clojure" "lisp"]}

Post yang [lepas](/posts/2018-05-21-refactor-core-async-menggunakan-transducer/)
ada kesilapan yang sangat kritikal. Cara penggunaan transducer dengan core.async
tidak salah. Yang salahnya ialah saya create semua `chan` yang untuk process
untuk setiap request. Maksudnya tak ada beza sama ada saya tulis secara
synchronous ataupun asynchronous.

> Load testing yang menunjukkan server Java mampu untuk handle request walaupun
> banyak silap yang saya buat menunjukkan betapa hebatnya Java Virtual Machine.

Semalam saya tengok [talk](http://www.youtube.com/watch?v=enwIIGzhahw) dari
Timothy Baldridge mengenai core.async. Yang membuatkan saya sedar yang saya
silap apabila beliau bercakap mengenai
[mult](https://clojuredocs.org/clojure.core.async/mult). Function `mult` itulah
yang saya tercari-cari.

Satu lagi yang saya terlepas pandang ialah
[go-loop](https://clojuredocs.org/clojure.core.async/go-loop). Clojure
menyediakan `go-loop` hanya untuk membuatkan loop di dalam block go. Maksudnya
cara tersebut selalu diguna pakai sehingga mereka memerlukan macro untuk proses
tersebut. Saya selalu dengar mengenai `event loop` setiap kali berbincang
mengenai asynchronous. Jadi, `go-loop` tersebut samalah seperti `event loop`
untuk dispatch dari channel ke channel.

Maka saya pun tulis semula code tersebut menggunakan `go-loop` dan `mult`.
Hasilnya,

```clojure
(def register-response-chan (chan))


(def validate-chan (chan 1 (map v/validate-registration)))

(def valid-form-chan (chan))
(def valid-form-mult (mult valid-form-chan))

(go-loop []
  (let [[errors form] (<! validate-chan)]
    (if errors
      (>! register-response-chan (-> (redirect "/daftar")
                                     (flash {:errors errors :data form})))
      (>! valid-form-chan form))
    (recur)))


(def username-existed-chan (chan))
(def email-existed-chan (chan))
(def user-existed-chan (chan))

(let [c (chan 1)]
  (tap valid-form-mult c)
  (async/thread
    (loop []
      (let [user (<!! c)]
        (>!! username-existed-chan
             (boolean (users/find-user-by-username db-spec user)))
        (recur)))))

(let [c (chan)]
  (tap valid-form-mult c)
  (async/thread
    (loop []
      (let [user (<!! c)]
        (>!! email-existed-chan
             (boolean (users/find-user-by-email db-spec user)))
        (recur)))))

(go-loop []
  (let [username-existed (<! username-existed-chan)
        email-existed (<! email-existed-chan)]
    (>! user-existed-chan (or username-existed email-existed))
    (recur)))


(def register-chan (chan))
(def register-mult (mult register-chan))

(let [c (chan)]
  (tap valid-form-mult c)
  (go-loop []
    (let [user-existed (<! user-existed-chan)]
      (if user-existed
        (>! register-response-chan (-> (redirect "/daftar")
                                       (flash {:message user-existed-msg})))
        (>! register-chan (<! c)))
      (recur))))


(def hashed-password-chan (chan))
(def email-verification-token-chan (chan))
(def prepared-user-chan (chan))

(let [c (chan)]
  (tap register-mult c)
  (async/thread
    (loop []
      (let [{:keys [password]} (<!! c)]
        (>!! hashed-password-chan (hashers/derive password))
        (recur)))))

(let [c (chan)]
  (tap register-mult c)
  (go-loop []
    (<! c)
    (>! email-verification-token-chan (-> (nonce/random-bytes 16)
                                          (codecs/bytes->hex)))
    (recur)))

(let [c (chan)]
  (tap register-mult c)
  (go-loop []
    (let [user (<! c)
          password (<! hashed-password-chan)
          token (<! email-verification-token-chan)]
      (>! prepared-user-chan (assoc user :password password :token token))
      (recur))))


(def persisted-user-chan (chan))
(def persisted-user-mult (mult persisted-user-chan))

(async/thread
  (loop []
    (let [user (<!! prepared-user-chan)
          row-count (users/register-user db-spec user)]
      (if (zero? row-count)
        (>!! register-response-chan (-> (redirect "/daftar")
                                        (flash {:message failed-msg})))
        (>!! persisted-user-chan user))
      (recur))))

(let [c (chan)]
  (tap persisted-user-mult c)
  (async/thread
    (loop []
      (let [user (<!! c)]
        (mailer/send-email-verification user)
        (recur)))))

(let [c (chan)]
  (tap persisted-user-mult c)
  (go-loop []
    (<! c)
    (>! register-response-chan (-> (redirect "/login")
                                   (flash success-msg)))
    (recur)))

(defn register [{:keys [params]} respond _]
  (let [form (select-keys params [:username :email :password])]
    (put! validate-chan form)
    (go (let [[res _] (alts! [register-response-chan (timeout 10000)])]
          (respond res)))))
```

Tengok function untuk handler yang bawah sekali boleh nampak dia takde create
channel atau thread yang baru, tugas dia hanya masukkan data ke channel yang
mula-mula iaitu `validate-chan` kemudian tunggu response dari
`register-response-chan`.

Kalau sesiapa yang mula dengar pasal `go loop` mungkin akan ingat, "Eh, takpe ke
buat banyak-banyak loop? Tak makan CPU ke nanti?". Jawapannya, sebab block go
ialah sebuah macro. Apabila kita buat panggilan `>!` atau `<!` dia bukannya
block, tetapi parking. Kalau tengok code untuk macro go, di dalamnya ada macam
state machine. Senang cerita jangan risau pasal performance.

Jadi, tengok balik code yang baru di atas, kalau sekali pandang macam nak
muntah. Berterabur saya letak channel. Waktu saya menulis code tersebut, saya
rasa macam saya sedang menulis code pub/sub atau reactive programming. Saya pun
google 'core.async pub/sub'. Lahaii, memang ada.
[Pub/sub](https://github.com/clojure/core.async/wiki/Pub-Sub) merupakan cara
yang lebih high level untuk menggantikan penggunaan `mult`.

Selepas refactor, code jadi begini:

```clojure
(def pub-chan (chan))
(def publication (pub pub-chan :msg-type))

(def response-chan (chan))
(sub publication :response response-chan)

(let [raw-input-chan (chan)]
  (sub publication :raw-input raw-input-chan)
  (go-loop []
    (let [{:keys [form]} (<! raw-input-chan)
          [errors valid-form] (v/validate-registration form)]
      (if errors
        (>! pub-chan {:msg-type :response
                      :response (-> (redirect "/daftar")
                                    (flash {:errors errors :data form}))})
        (>! pub-chan {:msg-type :valid-form :form valid-form}))
      (recur))))

(let [valid-form-chan (chan)]
  (sub publication :valid-form valid-form-chan)
  (async/thread
    (loop []
      (let [{:keys [form]} (<!! valid-form-chan)
            existing-username (users/find-user-by-username db-spec form)]
        (>!! pub-chan
             {:msg-type :existing-username :existing-username existing-username})
        (recur)))))

(let [valid-form-chan (chan)]
  (sub publication :valid-form valid-form-chan)
  (async/thread
    (loop []
      (let [{:keys [form]} (<!! valid-form-chan)
            existing-email (users/find-user-by-email db-spec form)]
        (>!! pub-chan {:msg-type :existing-email :existing-email existing-email})
        (recur)))))

(let [existing-username-chan (chan)
      existing-email-chan (chan)]
  (sub publication :existing-username existing-username-chan)
  (sub publication :existing-email existing-email-chan)
  (go-loop []
    (let [{:keys [existing-username]} (<! existing-username-chan)
          {:keys [existing-email]} (<! existing-email-chan)
          user-existed (boolean (or existing-username existing-email))]
      (>! pub-chan {:msg-type :whether-user-existed :user-existed user-existed})
      (recur))))

(let [whether-user-existed-chan (chan)
      valid-form-chan (chan)]
  (sub publication :whether-user-existed whether-user-existed-chan)
  (sub publication :valid-form valid-form-chan)
  (go-loop []
    (let [{:keys [user-existed]} (<! whether-user-existed-chan)
          {:keys [form]} (<! valid-form-chan)]
      (if user-existed
        (>! pub-chan {:msg-type :response
                      :response (-> (redirect "/daftar")
                                    (flash {:message user-existed-msg}))})
        (>! pub-chan {:msg-type :new-user :user form}))
      (recur))))

(let [new-user-chan (chan)]
  (sub publication :new-user new-user-chan)
  (async/thread
    (loop []
      (let [{:keys [password]} (:user (<!! new-user-chan))
            hashed-password (hashers/derive password)]
        (>!! pub-chan {:msg-type :hashed-password :password hashed-password})
        (recur)))))

(let [new-user-chan (chan)]
  (sub publication :new-user new-user-chan)
  (go-loop []
    (<! new-user-chan)
    (let [token (codecs/bytes->hex (nonce/random-bytes 16))]
      (>! pub-chan {:msg-type :email-verification-token :token token}))
    (recur)))

(let [hashed-password-chan (chan)
      email-verification-token-chan (chan)
      new-user-chan (chan)]
  (sub publication :hashed-password hashed-password-chan)
  (sub publication :email-verification-token email-verification-token-chan)
  (sub publication :new-user new-user-chan)
  (go-loop []
    (let [{:keys [user]} (<! new-user-chan)
          {:keys [password]} (<! hashed-password-chan)
          {:keys [token]} (<! email-verification-token-chan)
          prepared-user (assoc user :password password :token token)]
      (>! pub-chan {:msg-type :prepared-user :user prepared-user})
      (recur))))

(let [prepared-user-chan (chan)]
  (sub publication :prepared-user prepared-user-chan)
  (async/thread
    (loop []
      (let [{:keys [user]} (<!! prepared-user-chan)
            row-count (users/register-user db-spec user)]
        (if (zero? row-count)
          (>!! pub-chan {:msg-type :response
                         :response (-> (redirect "/daftar")
                                       (flash {:message failed-msg}))})
          (>!! pub-chan {:msg-type :persisted-user :user user}))
        (recur)))))

(let [persisted-user-chan (chan)]
  (sub publication :persisted-user persisted-user-chan)
  (async/thread
    (loop []
      (let [{:keys [user]} (<!! persisted-user-chan)]
        (mailer/send-email-verification user)
        (recur)))))

(let [persisted-user-chan (chan)]
  (sub publication :persisted-user persisted-user-chan)
  (go-loop []
    (<! persisted-user-chan)
    (>! pub-chan {:msg-type :response
                  :response (-> (redirect "/login")
                                (flash success-msg))})
    (recur)))

(defn register [{:keys [params]} respond _]
  (let [form (select-keys params [:username :email :password])]
    (go (let [[res _] (alts! [response-chan (timeout 10000)])]
          (respond (:response res))))
    (put! pub-chan {:msg-type :raw-input :form form})))

```

Akhirnya code tersebut jadi lebih kurang sama sahaja seperti penggunaan library
RxJava atau pub/sub yang lain.

Satu trick yang saya suka buat, saya letakkan transducer untuk log data yang
lalu ikut channel publisher. Dengan mudah dapat log semua event hanya dengan
menukar satu line.
