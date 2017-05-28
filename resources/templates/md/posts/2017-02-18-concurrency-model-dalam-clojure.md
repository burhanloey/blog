{:title "Concurrency Model dalam Clojure"
 :layout :post
 :tags  ["concurrency" "clojure"]}
 
 Selepas beberapa minggu menulis *fully asynchronous code* dalam Java 8, terdetik hati untuk menulis *asynchronous code* dalam programming language kegemaran saya iaitu Clojure.

Setelah menggodek internet, saya mendapati Clojure ada beberapa model untuk membuat *concurrency*, antaranya:

* future/promise
* coroutine (menggunakan library [core.async](https://github.com/clojure/core.async))
* actor model (menggunakan library [pulsar](https://github.com/puniverse/pulsar)/quasar)
* [promesa](https://github.com/funcool/promesa) (library untuk menggunakan CompletableFuture Java 8)
* parallel workers (interop dengan Java)

**Future/promise** sememangnya sudah ada secara *native* dalam Clojure. Saya memang suka model yang ini terutamanya apabila Clojure menggunakan *immutable data* secara *default* jadi saya tidak perlu bimbang dengan *race condition*. **Atoms**, **agents**, dan **refs** boleh digunakan untuk menguruskan *state*, jadi saya tidak perlu menggunakan *lock* seperti Java 7.

**Coroutine** adalah model yang sama digunakan dalam programming language Go. Model ini menghantar proses melalui *channel*.

**Actor model** pula digunakan dalam programming language seperti Scala dan Erlang. Saya tidak tahu sangat tentang actor model.

Library **promesa** membolehkan programmer Clojure menulis code seperti Java 8, tetapi saya cuba elakkan kerana perlu menggunakan library luar.

Untuk **parallel workers**, nasihat saya cuba elakkan.

Setelah melihat beberapa pilihan ini, saya menyangkakan pilihan cuma bergantung kepada programmer sahaja, ikut mana yang diminati. Namun, saya ada terbaca di stackoverflow yang menyatakan untuk memilih concurrency model perlulah mengikut situasi. Cuma saya masih belum nampak lagi.
