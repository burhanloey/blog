{:title "Penggunaan high level API core.async"
 :layout :post
 :tags  ["concurrency" "clojure" "lisp"]}

![Tweet dari Bruce Hauman](/images/bruce_core_async.png)

Semalam saya nampak Bruce Hauman ada
[tweet](https://twitter.com/bhauman/status/1000070247863365632) mengenai
core.async. Macam tau-tau je saya pun tengah mula nak guna. Saya pun tengok
video yang beliau tweet, [video](https://www.youtube.com/watch?v=096pIlA3GDo)
dari Timothy Baldridge yang menunjukkan bagaimana cara menggunakan core.async
dalam situasi sebenar. Beliau pun perasan ramai yang buat silap termasuklah
saya.

Dalam video tersebut, Timothy Baldridge sangat menggalakkan penggunaan function
`pipeline`. Sebelum ini saya masih tak faham waktu bila yang sesuai untuk
menggunakan function tersebut. Saya pun tengoklah
[documentation](https://clojure.github.io/core.async/) core.async.

Dalam documentation tersebut, ada banyak lagi function yang bunyinya lebih
kurang sama macam function untuk functional programming, antaranya map, reduce,
merge, dan split. Saya pun go through documentation untuk semua function
tersebut perlahan-lahan.

Semasa membaca documentation, saya perasan ada persamaan antara
function-function tersebut, iaitu ayat yang berbunyi lebih kurang seperti ini:

> The channels will close after the source channel has closed.

Maksudnya kalau kita hanya menggunakan function-function high level tersebut
untuk compose, sebaik sahaja kita `close!` channel input, automatik
channel-channel lain akan ditutup.

Berbeza dengan yang sebelum ini di mana saya terpaksa `close!` channel satu per
satu, dan secara jujurnya saya ada terlepas pandang untuk tutup beberapa
channel.

Jadi, selepas refactor, code saya jadi begini:

```clojure
(defn- check-existing-user [[_ form] out-ch]
  (->> [(async/thread (users/find-user-by-username db-spec form))
        (async/thread (users/find-user-by-email db-spec form))]
       (async/merge)
       (async/reduce #(boolean (or %1 %2)) false)
       (async/pipeline 1 out-ch (map #(vector % form)))))

(defn- generate-token []
  (codecs/bytes->hex (nonce/random-bytes 16)))

(defn- persist-user [[_ user] out-ch]
  (let [prepared-user (-> user
                          (update :password hashers/derive)
                          (assoc :token (generate-token)))]
    (->> (async/thread (users/register-user db-spec prepared-user))
         (async/pipeline 1 out-ch (map #(vector (zero? %) prepared-user))))))

(defn- split-if-error [ch]
  (async/split first ch))

(defn register-handler [{:keys [params] :as req} respond _]
  (let [input-ch                        (chan 1 (map v/validate-registration))
        [invalid-ch valid-ch]           (split-if-error input-ch)
        availability-ch                 (chan)
        [not-available-ch available-ch] (split-if-error availability-ch)
        persisting-ch                   (chan)
        [failed-ch success-ch]          (split-if-error persisting-ch)]
    (->> valid-ch
         (async/pipeline-async 1 availability-ch check-existing-user))
    (->> available-ch
         (async/pipeline-async 1 persisting-ch persist-user))

    (put! input-ch params)

    (go
      (respond
       (alt!
         invalid-ch
         ([result] (-> (redirect "/daftar")
                       (flash {:errors (first result) :data (second result)})))

         not-available-ch
         ([]       (-> (redirect "/daftar")
                       (flash {:message (:user-existed msg)})))

         failed-ch
         ([]       (-> (redirect "/daftar")
                       (flash {:message (:failed msg)})))

         success-ch
         ([result] (do
                     (future (mailer/send-email-verification (second result)))
                     (-> (redirect "/login")
                         (flash {:message (:success msg)}))))))
      (close! input-ch))))
```

Jika diperhatikan, saya hanya menggunakan satu block go sahaja. Tiada langsung
take (`<!`) atau put (`>!`). Saya juga hanya perlu `close!` satu channel sahaja
selepas selesai menghantar response. Error handling juga saya bawa sehingga ke
bahagian akhir sekali, jadi tiada *side-effect*.
