## Neden override sözcüğünü kullanmak zorundasın ? Paranormal Kod 

Derleyicileri kızdırmak, şikayetlenmelerini sağlamak bir çok durum için run-time sırasında ağlamaktan daha iyi bir alternatif. Aynı code-base içerisinde çalıştığınız diğer kişiler içinse, haberin olsun şu an bir şeyler yaptığın satıra ait fonksiyonun bulunduğu sınıfın bir atası var ve sahip olduğu bu atanın içerisinde bu fonksiyonun bir imzası var. Bu imzayı koruman gerekiyor.

`Base` ve `Derived` hiyerarşisine sahip bir yapımız olduğunu düşünelim. Fakat `Derived` sınıf içerisinde bulunan `msgHandle` fonksiyonun imzası `Base` sınıftan daha farklı olsun ve `override` niteleyicisini de eklemeyelim. `nobody's gonna know how they gonna know` yaklaşımı ile kod yazıyoruz. 


```cpp
struct Base
{
  virtual void msgHandle(uint64_t type)
  {
    std::cout << "Handled from Base" << std::endl;
  }
};

struct Derived : Base
{
  virtual void msgHandle(uint32_t type)
  {
    std::cout << "Handled from Derived" << std::endl;
  }
};

```

`x86-64 gcc 14.2` ile derleme yapıyoruz ve `vtable`'ı görebilmek için dump alalım. 

```cpp
...
...
...

vtable for Derived:
        .quad   0
        .quad   typeinfo for Derived
        .quad   Base::msgHandle(unsigned long)
        .quad   Derived::msgHandle(unsigned int)
vtable for Base:
        .quad   0
        .quad   typeinfo for Base
        .quad   Base::msgHandle(unsigned long)
typeinfo for Derived:
        .quad   vtable for __cxxabiv1::__si_class_type_info+16
        .quad   typeinfo name for Derived
        .quad   typeinfo for Base
typeinfo name for Derived:
        .string "7Derived"
typeinfo for Base:
        .quad   vtable for __cxxabiv1::__class_type_info+16
        .quad   typeinfo name for Base
typeinfo name for Base:
        .string "4Base"
```



`Derived` sınıfının 2 farklı fonksiyon gösterecisine sahip olduğunu görebiliriz. Peki sorun nerede başlıyor ? 

`virtual` olarak nitelendirilmiş bir fonksiyonu farklı imza yapısına sahip bir şekilde fakat doğru şekilde çağrılmasını sağlamak yukardaki yaklaşım ile mümkün değil. Bunun için belki `CRTP`'den destek alabilirsin. Ama burada bulunan problem bundan daha fazlası. 

Çoğu zaman çalıştığımız şirketlerde sıfırdan bir şeyler inşa etmiyoruz. Bazen kullandığımız ara katmanda güvenlik açığından dolayı güncel versiyonu port etmemiz gerekiyor, bazen varolan tasarımı kabul edip onun üzerine yeni kat çıkıyoruz. Bazen ihtiyaca yönelik mevcut ko vdu güncelliyoruze en önemlisi kod-base'e gerçekten aktif olamıyoruz. Çünkü onlarca farklı komponent farklı uzmanlık alanları olan kişiler tarafından geliştiriliyor. Hakim olmaya başladığında da yeni bir iş aramaya başlıyorsun :sweat_smile:

Gerçek bir senaryodan bahsedeyim, kullanılan bir komponentin yeni bir versiyonuna geçiş kararı veriliyor. Ekip bir araya geliyor ve port işlemini gerçekleştiyor. Güncellenen kütüphanede fonksiyonların parametrik yapısında `UNSIGNED` ifadesinden `UINT64_t` bir geçiş olduğu için, üst katmanda bulunan bazı fonksiyonlarında buna adapte edilmesi gerekiyor. İşte tam da burada sorun başlıyor, çünkü sınıf hiyerarşileri çok katmanlı ve bazı yerlerde `override` niteliyicisi kullanılmamış. Yani yapılması gereken bazı değişiklikler atlanıyor, çünkü o yükü derleyiciden alıp kullanıcıya vermiş oluyorsun. Kod derleniyor ve durum halting. 


```diff
... 
... 
...
struct Derived : Base
{
- virtual void msgHandle(uint32_t type) 
+ virtual void msgHandle(uint32_t type) override
  {
    std::cout << "Handled from Derived" << std::endl;
  }
};

*/
```

Yukarda bulunan kod için `override` niteleyicisi eklendiği takdirde derleyici carlayacaktı. Çünkü `Derived` sınıf kendi atasından olmayan bir şeyleri `override` etmeye çalışıyor. 

### Best Practice
`commercial or professional procedures that are accepted or prescribed as being correct or most effective.`


[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/vvolkanunas)










