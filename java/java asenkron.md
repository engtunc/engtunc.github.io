
- [Java 8 – CompletableFuture ile Asenkron Programlama](#java-8-%E2%80%93-completablefuture-ile-asenkron-programlama)
  - [Syncronous vs. Asyncronous](#syncronous-vs-asyncronous)
  - [CompletableFuture#runAsync](#completablefuturerunasync)
  - [CompletableFuture#anyOf](#completablefutureanyof)
  - [CompletableFuture#supplyAsync](#completablefuturesupplyasync)
  - [CompletableFuture#runAfterBoth*](#completablefuturerunafterboth)
  - [CompletableFuture#handle*](#completablefuturehandle)





## Java 8 – CompletableFuture ile Asenkron Programlama
Tarih : 18 Kasım 2014

CompletableFuture sınıfı, Java 8 içerisinde asenkron operasyonlar için özelleştirilen bir sınıftır. Java ortamında Java SE ve Java EE teknolojilerinde bir çok asenkron programlama imkanı halihazırda geliştiricilere sunulmaktadır. CompletableFuture sınıfı ise, asenkron programla ihtiyaçlarına çok daha genel çözümler getirmektedir.

### Syncronous vs. Asyncronous

Eğer bir uygulamanın akışında, bir görevin başlaması diğer görevin bitişine bağlı ise, buna senkron programlama; Eğer bir görevin başlaması diğer görevin başlamasına engel olmuyorsa da asenkron programlama kavramları ortaya çıkmaktadır. Java programlama dili asenkron programlamaya çoğu noktada imkan sağlamakla birlikte, dilin genel yatkınlığı çoğu dil gibi senkron programlama yönündedir. Fakat, örneğin JavaScript gibi bir dili incelediğinizde, asenkronitinin dilin diyaznını ne derece etkilediğini gözlemleyebilirsiniz.
Örneğin, elimizde fetchFromDatabase ve saveFiles metodları olduğunu varsayalım. İlk metodun koşturulma süresi 5, diğerinin ise 3 saniye alıyor olsun.

```java

private List<String> fetchFromDatabase(){
...
Thread.sleep(5000);
...
}

private List<byte[]> readFiles(){
...
Thread.sleep(3000);
...

}
```


Şimdi bu iki metodu peşisıra koşturalım.

fetchFromDatabase();readFiles();

Bu iki görevin tamamlanma süresi ne kadar olacak?

Cevap: Math.sum(5,3) = 8

Java dilinin genel doğası gereği bu iki iş sırasıyla işletilecektir. Fakat dikkat edilirse, yapılan iki iş birbirinden tamamen bağımsızdır. Biri DB’den veri çekiyor, diğeri ise dosyalama sisteminden dosya okuyor. Dolayısıyla, bu işlerden birinin başlaması için diğer işin tamamlanması beklenmek zorunda değil.
Bu iki metodun asenkron olarak çalışması için geleneksel çokişlemcikli programlama ile harici asenkron iş kolları oluşturulabilir. Fakat, burada geleneksel yöntemlerin dışında CompletableFuture nesnesi üzerinden gitmekte fayda görüyorum.


```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
...
}
```

CompletableFuture sınıfı Future ve CompletionStage arayüzleri türünden jenerik bir sınıf. CompletableFuture türünden nesneler, nesnenin yapılandırıcısı üzerinden veya CompletableFuture ‘nin çeşitli statik metodlarıyla oluşturulabilmektedir.

CompletableFuture ile doğası senkron koşmak olan bir işi, asenkron koşar hale getirebilirsiniz. Aslında yapılan iş, senkron koşan işin arka plana itilerek koşturulması ve mevcut program akışının kesintiye uğratılmamasıdır. CompletableFuture nesneleri, ekstra olarak tanımlanmadığı sürece tek bir ForkJoin Thread havuzu ile işlerini asenkron olarak arka planda koşturmaktadır.
Şimdi yukarıdaki senkron örneği asenkron hale getirelim. Bunun için CompletableFuture#runAsync metodu kullanılabilir.

```java
public static CompletableFuture<Void> runAsync(Runnable runnable) {
...
return f;
}

```

### CompletableFuture#runAsync

CompletableFuture#runAsync metodu Runnable türünden bir görev sınıfı kabul etmektedir, arından CompletableFuture türünden bir nesne döndürmektedir. Parametre olarak iletilen Runnable nesnesi, arkaplanda asenkron olarak koşturulmaktadır.

NOTE : Runnable arayüzü tek bir soyut metoda sahip olduğu için, Lambda fonksiyonu olarak temsil edilebilir. () → { }

```java

CompletableFuture<Void> futured1 = CompletableFuture.runAsync(() -> {

fetchFromDatabase(); (1)

});

CompletableFuture<Void> futured2 = CompletableFuture.runAsync(() -> {

saveToFile(); (2)

});

futured1.join(); (3)
futured2.join(); (4)

```


Yukarıdaki (1) ve (2) numaralı işler bu noktadan sonra arkaplanda ForkJoin thread havuzu içinde koşturulmuş olacak. Böylece (2) numaralı iş, (1) numaralı iş koşturulmaya başlatıldıktan hemen sonra çalışmaya başlayacak, diğerinin işe koyulmasını bloke etmeyecek.
Peki şimdi bu iki asenkron görevin tamamlanma süresi ne kadar olacak?
Cevap: Math.max(5,3) = 5
Burada iki iş birden hemen hemen aynı anda başlayacağı için, iki işin toplamda tamamlanma süresi yaklaşık olarak en fazla süren görev kadar olacaktır.
NOTE
CompletableFuture#join metodu, asenkron olarak koşturulan görev tamamlanana kadar, uygulama akışının mevcut satırda askıda kalmasını sağlar. Yani (3)  ve (4) satırlarından sonraki satırlarda, yukarıdaki iki işin birden tamamlanmış olduğunu garanti edebiliriz.
CompletableFuture#allOf
Birden fazla CompletableFuture nesnesini birleştirir. Ancak herbir iş birden tamamlandığında, CompletableFuture nesnesi tamamlandı bilgisine sahip olur.

public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {

...

}

Örneğin;

```java
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
...
Thread.sleep(5000);
...

System.out.println("İlk görev tamamlandı..");
});

CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
...
Thread.sleep(15000);
...

System.out.println("Diğer görev tamamlandı..");
});

CompletableFuture<Void> allOf = CompletableFuture.allOf(future1, future2);

System.out.println("Bir arada iki derede.");

allOf.join();

System.out.println("Bitti.");
```


Yukarıda iki tane asenkron iş koşturulmaktadır. Bir tanesi 5, diğeri ise 15 saniye sürmektedir. Eğer asenkron koşan uygulama akışında, bu iki iş bitene kadar bir noktada beklemek istiyorsak, CompletableFuture#allOf dan faydalanabiliriz. Uygulama akışının askıda bekletilmesi ise CompletableFuture#join metodu ile sağlanmaktadır.

```java
Çıktı
Bir arada iki derede. // 0. saniyede
İlk görev tamamlandı.. // 5. saniyede
Diğer görev tamamlandı.. // 15. saniyede
Bitti. // 15. saniyede
```

### CompletableFuture#anyOf

Birden fazla CompletableFuture nesnesini birleştirir. Herhangi bir görev tamamlandığında, CompletableFuture nesnesi tamamlandı bilgisine sahip olur.
Örneğin;
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
...
Thread.sleep(5000);
...

System.out.println("İlk görev tamamlandı..");
});

CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
...
Thread.sleep(15000);
...

System.out.println("Diğer görev tamamlandı..");
});

CompletableFuture<Void> anyOf = CompletableFuture.anyOf(future1, future2);

System.out.println("Bir arada iki derede.");

anyOf.join();

System.out.println("Bitti.");
Çıktı
Bir arada iki derede. // 0. saniyede
İlk görev tamamlandı.. // 5. saniyede
Bitti. // 5. saniyede
Diğer görev tamamlandı.. // 15. saniyede

### CompletableFuture#supplyAsync

CompletableFuture#supplyAsync metodu CompletableFuture#runAsync metodu gibidir. Fakat koşma sonucunda geriye bir sonuç döndürebilmektedir. Bir iş sonunda geriye hesaplanmış bir değer döndürmeye ihtiyaç duyulduğu noktada kullanılabilir.
Örneğin, /var/log dizinindeki tüm dosya ve klasörlerin listesini hesaplatmak istiyoruz diyelim.
CompletableFuture<List<Path>> future = CompletableFuture.supplyAsync(() -> {
Stream<Path> list = Stream.of();

try {
list = Files.list(Paths.get("/var/log"));
} catch (IOException e) {
e.printStackTrace();
}

return list.collect(Collectors.toList());

});
Bu ihtiyacı Files#list metodu ile sağlayabiliriz. Files#list metodu tanımlanan dizindeki tüm dizin ve dosyaları bir Path listesi olarak sunmaktadır. Dizindeki dosya ve dizin sayısına göre bir sonucun elde edilmesi belirli bir zaman gerektirebilir.

NOTE
CompletableFuture#supplyAsync metodu Supplier türünden bir nesne kabul ettiği için bir Lambda fonksiyonu olarak temsil edilebilirdir. () → T

CompletableFuture’in çoğu metodu işlerini asenkron olarak arkaplanda koşturmaktadır. Bu sebeple mevcut uygulamanın akışını askıda bırakmamaktadır.
Bir CompletableFuture’in iş bitimindeki sonucunu elde etmenin iki yöntemi bulunmaktadır.

İlk yol, join() metodu kullanmak

join() metodu, asenkron olarak işletilen görev tamamlanana kadar uygulama akışını askıda tutmaktadır. İş tamamlandığında ise varsa sonuç değerini döndürmektedir.

CompletableFuture<List<Path>> future = CompletableFuture.supplyAsync(() -> {
Stream<Path> list = Stream.of();

try {
list = Files.list(Paths.get("/var/log"));
} catch (IOException e) {
e.printStackTrace();
}

return list.collect(Collectors.toList());

});


// Varsa diğer işler bu arada yapılabilir


List<Path> liste = future.join(); (1)


// join() tamamlanana kadar buraya erişim devam etmez

1.	İş bitiminde elde edilen sonuç listesi

İkinci yol, thenAccept* metodu kullanmak
thenAccept metodu ile callback stilinde asenkron işlerin sonuçları elde edilebilir. thenAccept metodu Consumer<T> türünden bir nesne kabul etmekte ve sonucu onun üzerinden sunmaktadır.
CompletableFuture<List<Path>> future = CompletableFuture.supplyAsync(() -> {
Stream<Path> list = Stream.of();

try {
list = Files.list(Paths.get("/var/log"));
} catch (IOException e) {
e.printStackTrace();
}

return list.collect(Collectors.toList());

});

future.thenAccept( (List<Path> paths) -> {
// liste burada
});

Yukarıdaki thenAccept ile, CompletableFuture nesnesine bir hook tanımlanmış olur. İş bitiminde sonuç elde edildiği zaman bu metod otomatik olarak işletilir. Sonuç parametre olarak geliştiriciye sunulur.

### CompletableFuture#runAfterBoth*

İki asenkron iş birden tamamlandığında bir Runnable türünden nesneyi koşturmayı sağlar.

```java

CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
try {
Thread.sleep(5000);
} catch (InterruptedException e) {
e.printStackTrace();
}
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
return 10;
});

future1.runAfterBoth(future2,()->{
System.out.println("İkisi birden bitti"); // 5. saniyede
});
CompletableFuture#runAfterEither*
İki asenkron işden herhangi biri tamamlandığında bir Runnable türünden nesneyi koşturmayı sağlar.
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
try {
Thread.sleep(5000);
} catch (InterruptedException e) {
e.printStackTrace();
}
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
return 10;
});

future1.runAfterEither(future2,()->{
System.out.println("İkisinden biri tamamlandı.."); // 0. saniyede
});
```


### CompletableFuture#handle*

CompletableFuture#handleAsync metodu bir önceki asenkron görevin sonucunu işlemek ve ardındaki görevlere paslamak için yapılandırılmıştır. CompletableFuture#handleAsync ile, birbirini besleyen zincirler şeklinde asenkron iş akışları yazılabilir.

Örneğin, iki asenkron işten birini, diğerini besler şeklinde yapılandıralım.

Görev 1

Asenkron olarak bir dizindeki tüm dosya ve dizinler bulunsun

Görev 2

Bulunan dizinlerin boyut bilgisi asenkron olarak hesaplansın

Görev 3

Dosya yolu ve boyut bilgisi asenkron olarak listelensin.


```java

CompletableFuture.supplyAsync(() -> { (1)

Stream<Path> list = Stream.of();

try {
list = Files.list(Paths.get("/var/log"));
} catch (IOException e) {
throw new RuntimeException(e);
}

return list.collect(Collectors.toList());

}).handleAsync((paths, throwable) -> { (2)

Map<Path, Long> pathSizeMap = new HashMap<>();

try {
for (Path path : paths) {
long size = Files.size(path);
pathSizeMap.put(path, size);
}
} catch (IOException e) {
throw new RuntimeException(e);
}

return pathSizeMap;

}).thenAccept(map -> { (3)

for (Map.Entry<Path, Long> entry : map.entrySet()) {
System.out.printf("%s | %d bytes %n",entry.getKey(),entry.getValue());
}

});
```

1.	Dosya ve dizinleri liste olarak döndürür
2.	Elde ettiği listeden her bir dizinin boyutunu hesaplar, bir Map nesnesi olarak sunar.
3.	En son üretilen Map nesnesinden dosya yolu ve boyutunu birbir çıktılar.

CompletableFuture sınıfının Java’da asenkron programlamayı hiç olmadığı kadar kolaylaştırdığını söyleyebilirim.

Tekrar görüşmek dileğiyle.


