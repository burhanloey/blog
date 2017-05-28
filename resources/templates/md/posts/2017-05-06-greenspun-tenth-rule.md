{:title "Greenspun's Tenth Rule"
 :layout :post
 :tags  ["concurrency" "clojure" "completablefuture" "java" "lisp"]}
 
 Sebelum ini saya pernah menggunakan feature dari Java 8 untuk membuat *concurrency* iaitu dengan menggunakan interface [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html). Ada suatu ketika, interface tersebut bukannya memudahkan, tapi merumitkan lagi keadaan.

Masalah berpunca kerana Java 8 hanya menyediakan [BiFunction](https://docs.oracle.com/javase/8/docs/api/java/util/function/BiFunction.html) yang kita boleh gunakan untuk method [thenCombine](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-). Maksudnya method tersebut hanya boleh menerima dua proses sahaja. Jika kita mahu lebih daripada dua, kita boleh menggunakan library [Javaslang/Vavr](https://github.com/vavr-io/vavr) dan menggunakan interface Function3, Function4, dan sebagainya tetapi itu topik yang lain.

Untuk menyelesaikan proses serentak yang lebih daripada dua, Java 8 ada menyediakan method [allOf](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-). Namun, method tersebut pun ada karenahnya sendiri.

Yang pertama, parameter untuk **allOf** ialah *variadic*. Maksudnya method tersebut boleh menerima tak kira berapa banyak argument yang kita mahu letak, ataupun hanya letak array. Oops, tunggu dulu, kita tidak boleh meletakkan array untuk **allOf** sebab CompletableFuture ialah *generic interface*. Dalam Java kita tidak boleh membuat array untuk generic. Error tersebut dipanggil *generic array creation error*.

Yang kedua, method **allOf** akan return CompletableFuture dengan data type *Void*. Ya, *VOID*. Memang ramai yang complain pasal ini.

Jadi, untuk menggunakan **allOf** kita perlu menghantar argument satu per satu, kemudian untuk mendapatkan *result* kita ambik balik daripada Future masing-masing menggunakan [join](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#join--). Sesiapa yang dah boleh agak, ya, code tersebut memang akan jadi sangatlah buruk.

Sekarang kita patah balik ke programming language Clojure. Di post yang sama sebelum ini, saya juga ada perkenalkan library Clojure yang menggunakan CompletableFuture dari Java 8, iaitu library [promesa](https://github.com/funcool/promesa). Saya pun intai-intai macam mana library tersebut mengendalikan masalah ini.

Seperti yang saya jangkakan, [sangat simple, sangat mudah](http://funcool.github.io/promesa/latest/#working-with-collections). Ya rabbi, kedua-dua isu yang saya senaraikan di atas terus hilang.

Secara jujurnya saya memang tepuk dahi sewaktu membaca documentation tersebut.

Akhir kata, jika nak dikaitkan dengan tajuk artikel ini, saya masih setuju dengan pendapat [Greenspun's tenth rule](https://en.wikipedia.org/wiki/Greenspun%27s_tenth_rule) yang menyatakan:

```
Any sufficiently complicated C or Fortran program contains an ad-hoc,
informally-specified, bug-ridden, slow implementation of half of Common Lisp.
```

Cuma ganti C dengan Java, dan Common Lisp dengan Clojure sebab mereka adik-beradik.
