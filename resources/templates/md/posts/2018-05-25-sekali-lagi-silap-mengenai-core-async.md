{:title "Sekali lagi silap mengenai core.async"
 :layout :post
 :tags  ["concurrency" "clojure" "lisp"]}

Sekali lagi saya salah mengenai core.async dalam post saya yang lepas.

> Secara ringkasnya cara yang betul ialah yang mula-mula di mana saya
> menggunakan transducer. Cara yang salah ialah yang di mana saya menggunakan
> pub sub (yang saya sangkakan betul).

Semalam saya tidak berpuas hati dengan code untuk pub sub, sebab perlu tulis
terlalu panjang. Takkanlah perlu tulis code seperti itu walaupun untuk buat
benda yang remeh. Saya pun baca-bacalah mengenai core.async, async vs. sync, dan
motivasi di sebalik setiap paradigm.

Bila sebut tentang concurrency, masalahnya bukan mengenai bagaimana untuk
menjalankan code secara parallel (serentak), tetapi mengenai bagaimana untuk
menyatukan kembali data-data yang telah dihantar ke thread-thread yang
berlainan.

Antara teknik yang selalu kita dengar apabila sebut tentang asynchronous ialah
callback atau event-driven. Jadi ada paradigm seperti future/promise untuk
mengelakkan callback hell. Paradigm untuk core.async adalah seperti
BlockingQueue. Saya pun baca mengenai rationale di sebalik design core.async.

Yang saya faham, tujuan core.async adalah supaya kita boleh menulis code seperti
synchronous tetapi di sebalik tabir, code akan berjalan secara asynchronous.
Contohnya, apabila kita menggunakan take (`<!`) dalam block go, dalam kepala
kita, kita rasakan seperti itu operasi blocking, tetapi di sebalik tabir,
program menulis callback untuk sambung semula proses apabila channel tersebut
dah ready.

Untuk meyakinkan diri saya, saya tulis code yang sama untuk server Netty iaitu
http-kit untuk bandingkan dengan server Jetty yang saya guna sebelum ini. Kalau
load testing menunjukkan request per second untuk Netty lebih tinggi daripada
Jetty bermakna saya betul, kalau sama bermakna code saya salah.

Hasilnya,

```text
Code transducer untuk Jetty: 3500+ req/s
Code transducer untuk Netty: 8000+ req/s
Code pub/sub untuk Jetty: 2000+ req/s
Code pub/sub untuk Netty: 2000+ req/s
```

Apa yang berlaku dalam code pub/sub ialah semua request perlu melalui satu
channel sahaja. Saya dengan tidak sengaja telah mengubah multithreaded server
menjadi single-threaded, asynchronous server. Aduh.

Dari segi masa response, page untuk code pub/sub load lebih laju, tetapi setiap
kali saya request page, req/s jatuh ke 1500+. Untuk code transducer, masa untuk
load page dalam 1-2 saat, tetapi req/s tak jatuh-jatuh.

Sekarang dah yakin bahawa code transducer adalah yang betul. Cuma ada sedikit
kekurangan. Dalam code tersebut, saya terlupa untuk tutup channel selepas
menghantar response. Selepas sedikit refactor, code jadi begini:

```clojure
(defn- validate-form [[response-chan form]]
  (let [form-chan (chan)]
    (go
      (let [[errors _] (v/validate-registration form)
            valid? (not errors)]
        (if valid?
          (>! form-chan form)
          (do
            (close! form-chan)
            (>! response-chan (-> (redirect "/daftar")
                                  (flash {:errors errors :data form})))))))
    [response-chan form-chan]))

(defn- check-existing-user [[response-chan form-chan]]
  (let [user-chan (chan)]
    (go
      (if-let [form (<! form-chan)]
        (let [username-existed (thread (users/find-user-by-username db-spec form))
              email-existed (thread (users/find-user-by-email db-spec form))
              available? (not (or (<! username-existed)
                                  (<! email-existed)))]
          (if available?
            (>! user-chan form)
            (do
              (close! user-chan)
              (>! response-chan (-> (redirect "/daftar")
                                    (flash {:message (:user-existed msg)}))))))
        (close! user-chan)))
    [response-chan user-chan]))

(defn- persist-user [[response-chan user-chan]]
  (go
    (when-let [user (<! user-chan)]
      (let [email-verification-token (codecs/bytes->hex (nonce/random-bytes 16))
            prepared-user (-> user
                              (update :password hashers/derive)
                              (assoc :token email-verification-token))
            success? (->> (thread (users/register-user db-spec prepared-user))
                          <!
                          pos?)]
        (if success?
          (do
            (future (mailer/send-email-verification user))
            (>! response-chan (-> (redirect "/login")
                                  (flash {:message (:success msg)}))))
          (>! response-chan (-> (redirect "/daftar")
                                (flash {:message (:failed msg)}))))))))

(defn register-handler [{:keys [params] :as req} respond _]
  (let [response-chan (chan)
        form (select-keys params [:username :email :password])]
    (go
      (->> [response-chan form]
           validate-form
           check-existing-user
           persist-user))
    (go
      (respond (<! response-chan))
      (close! response-chan))))
```

Code yang terbaharu ini lain sedikit daripada yang transducer sebab dulu saya
kurang faham tujuan block go. Penggunaan transducer sama sahaja dengan
thread-last macro `->>` yang saya gunakan di sini.

Kalau saya tidak tutup channel, takut lama-lama server akan ada software rot.

Dengan code yang baru ini, server Jetty mampu capai 4200+ req/s. Mungkin sebab
kurang channel ataupun sebab saya tutup channel.

Apa-apa pun sekarang saya sangat berpuas hati sebab boleh buat server (http-kit)
sama performance dengan Play Framework, framework yang saya guna di tempat kerja
yang lepas.
