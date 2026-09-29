Pirates Match
====
#### Unity 6 ve Firebase ile geliştirdiğim, korsan temalı bir mobil match-3 oyunu.
**Projeyi çalıştırmak için [Başlarken](#başlarken) bölümündeki adımları izleyebilirsin.**

Giriş
------

Pirates Match, Unity ve C# ile geliştirdiğim, korsan temalı bir mobil match-3 oyunu. Bölümleri geçmek için taşları match eder, roket ve bomba oluşturur, bunları birleştirerek daha geniş alanları temizlersin. Hamlelerini dikkatli kullanman gerekir; her bölümün kendi board'u ve hedefleri vardır.

Oyun esnasında bir noktada özel bir hamleye ihtiyacın olursa hammer, bomb veya cannon SpecialStike'larını kullanabilirsin. Hammer, seçtiğin taşın çevresini artı şekilinde temizler; bomb, 3x3'lük alanı temizler; top ise seçtiğin satırı temizler. Her birinin kendine ait kısa bir animasyonu vardır.

Oyun dışında diğer oyuncularla bir klan kurabilir, diğer oyuncularla sohbet edebilir, can isteyebilir ve can bağışı yapabilirsin. İlerlediğin bölüm sayısı ve puanların, global rank'ta sıralamada da yerini belirler.

Kodun yapısı oynarken karşılaştığın kavramları izler: bölüm, board, hücre, taş, match ve cascade. Aşağıdaki açıklamalar bu parçaların ne yaptığını ve nasıl bir araya geldiğini anlatır.

### Motivasyon

Bir match-3 oyununda eşleşmeyi bulmak işin bir kısmı. Taşların nasıl düştüğü, bir patlamanın diğerini ne zaman tetiklediği ve oyunun dokunuşa ne kadar çabuk karşılık verdiği de en az bunun kadar önemli.

Pirates Match’i geliştirirken bu ayrıntılar üzerinde durdum. Taşlar düşerken hızlanır, yere inerken esneme animasyonu olur, swap sırasında arkalarında duman bırakır. Yeni bir swap yapmak için cascade'in bitmesini beklemen gerekmez.

Bu davranışlar arttıkça board kodu da büyüdü. Başlangıçta 2.200 satıra ulaşan sınıfı; doldurma, giriş, efektler ve özel taş zincirleri gibi ayrı işler üstlenen bileşenlere ayırdım. Bu düzenlemenin ayrıntılarını [tasarım dokümanında](docs/superpowers/specs/2026-09-21-gameboard-split-design.md) bulabilirsin.

Level design tamamen scriptible oject'ler ile hızlı ve basit bir şekilde yönetilir. Oyuncular arasında klan, sohbet ve can bağışı için Firebase backend servisi kullanılır.

------
Teknoloji
======
Unity
------

Oyunun temelinde **Unity 6000.5** ve 2D Universal Render Pipeline bulunur. Korsan karakterinin animasyonlarını Spine2D'den, patlama ve iz partical effect'leri Cartoon FX Remaster asset'inden sağlanır.

Oyun kodları `Match3`, editör araçları `Match3.Editor` assembly’sinde yer alır. Namespace’ler klasörlerle aynı düzeni izler. Örneğin board kodunu `Match3.Gameplay.Board`, menü kodunu `Match3.Menu`, Firebase servislerini `Match3.Backend` altında bulabilirsin.

### Board Design

Ekranda gördüğün board 6 sütun ve 8 satırdır. Bunun üzerinde, oyuncunun görmediği 7 satır daha bulunur. Yeni taşlar bu görünmeyen alanda oluşturulur.

Board'un şeklini ve tasarımını scriptiable object'lerden rahatca tasarlanabilir.

`BoardMask`, bu veriden oyun sırasında bir mesh üretir. Her hücre bir pikseldir: kapalı hücreler opak, açık hücreler şeffaf olur. Doku bir `SpriteMask` üzerinde kullanılır ve opak kısımlardaki taşlar gizlenir. Böylece her board şekli için ayrı bir çerçeve çizmek gerekmez.

### Hareket ve his

Taşlar düşüş sırasında düşüş hızları zamanla artar, bu da oyun içinde daha doğal bir hissiyat yaratır.

Taşlar yere ulaştığında ise tatlı bir esneme animasyonuyla güzel bir görünüm yakalanır.

### Eşzamanlı çözümleme

Oyun içinde hareket eden taşlar olmasında dahi hareket etmeyen taşları swap yapabilir ve match edebilirsin; bu sayede cascade'in bitmesi oyuncuya bekletilmeden, oyuncuya anında geri dönüş yapılır ve akıcı bir hissiyat yakalanır.

Birden fazla işlemin birlikte ilerlemesi iki kurala dayanır:
- Board'u değiştiren kod, coroutine'in ilk `yield`'inden önce biter.
- Her çözümleme kendi zincir durumunu (`ChainContext`) taşır.

Her cascade kendi durumunu taşır, dolayısıyla bir cascade'in işlemi diğerini etkilemez. `IsMoving` durumundaki taşlar da eşleşme ve takas kontrollerine katılmaz.

### Proje yapısı

```
Assets/
├─ Scripts/
│  ├─ Gameplay/
│  │  ├─ Board/      Board modeli, eşleşme, doldurma ve havuz, zincirler, giriş, maske
│  │  ├─ Potions/    Taş bileşeni ve özel taş görselleri
│  │  ├─ Strikes/    Hammer / Bomb / Cannon special strike'ları
│  │  └─ Session/    Bölüm kuralları, HUD ve oyun sonu panelleri
│  ├─ Levels/        Bölüm verisi, katalog ve bölüm asset'leri
│  ├─ Menu/          Ana menü: sayfalar, profil, clan, sohbet, sıralama, ayarlar
│  ├─ Backend/       Firebase başlatma, clan ve sohbet servisleri, Firestore modelleri
│  ├─ Shared/        Sahne adları, ses ayarları
│  └─ Editor/        Bölüm şekli editörü
├─ Scenes/           MainMenu, GameBoard
└─ Prefabs/          Taşlar, special strike'lar, menü satırları
docs/                Tasarım dokümanları ve Firebase notları
firestore.rules      Firestore güvenlik kuralları
```

Firebase
------

Hesap işlemlerini **Firebase Authentication**, veri saklamayı **Firestore** üstlenir. Projede Firebase Unity SDK 13.15.0 kullanılır.

Oyuncunun başlamadan önce kayıt olması gerekmez. İlk açılışta anonim bir hesap oluşturulur; cihazdaki oturum bilgileri korunduğu sürece sonraki açılışlarda aynı hesapla devam edilir.

### Veri modeli

Oyuncu, klan ve mesaj verileri şu koleksiyonlarda tutulur:

| Yol | İçerik |
|---|---|
| `users/{uid}` | ad, avatar, en yüksek bölüm, toplam puan, bölüm rekorları, can, altın, clan |
| `clans/{clanId}` | ad, açıklama, amblem, lider, üye sayısı, toplam puan, katılım kuralları |
| `clans/{clanId}/messages/{id}` | sohbet mesajları ve can istekleri |
| `clanNames/{nameLower}` | clan adlarını benzersiz tutan isim rezervasyonu |

İki klanın aynı adı kullanmasını önlemek için `clanNames` içinde bir isim rezervasyonu tutulur. Klan oluşturulurken bu kayıt da işlemin bir parçasıdır.

### Transaction'lar

Bir klan kurduğunda yalnızca yeni bir klan kaydı oluşmaz. Oyuncunun üyeliği değişir, altını azalır ve klan adı rezerve edilir. Bu değişikliklerin birlikte tamamlanması için transaction kullanılır:

- **Klan kurma:** İsim rezervasyonu, klan kaydı, oyuncunun üyelik bilgisi ve altın kesintisi aynı işlemde kaydediliyor.
- **Klana katılma:** Kapasite ve mevcut üyelik, sunucudaki güncel veriler üzerinden kontrol ediliyor. Böylece katılma butonuna iki kez basılması üye sayısını iki kez artırmıyor.
- **Klandan ayrılma:** Ayrılan kişi liderse görev en yüksek seviyeli üyeye devrediliyor. Devir sırasında bu oyuncunun hâlâ klan üyesi olduğu tekrar kontrol ediliyor.

### Can yenilenmesi ve bağış

Oyuncu oyunu kapattığında canların yenilenmesi için çalışan bir sayaç gerekmez. Can sayısı ve son güncelleme zamanı saklanır. Oyuna dönüldüğünde geçen süre hesaplanır ve hak edilen canlar eklenir.

Can bağışında ise küçük bir ayrıntı var: bir oyuncu, başka bir oyuncunun belgesine yazamaz. Bu yüzden bağış yapan kişi isteğe yalnızca kendi kimliğini ekler. Canı isteyen oyuncu bu bağışı kendi hesabına aktarır ve isteği kapatır. Son iki değişiklik aynı batch içinde kaydedilir.

Sohbet mesajları, 7 günlük süre için ayarlanmış Firestore TTL politikasıyla temizlenir. Sıralamadaki yerini bulmak için de bütün oyuncu belgelerini indirmek gerekmez; bunun için bir `count()` sorgusu kullanılır.

### Güvenlik kuralları

Veriye kimin erişebileceği ve hangi değerleri yazabileceği [`firestore.rules`](firestore.rules) içinde tanımlıdır:

- Oyuncu yalnızca kendi dokümanına yazabilir.
- Puan ve bölüm geri gidemez, can 0–5 arasında kalır.
- Sohbeti yalnızca o clanın üyeleri okuyabilir.
- Bağışçı istek mesajında yalnızca kendi kimliğini ekleyebilir.

------
Kavramlar
======

Pirate Match nesne yönelimli bir yapı kullanır. Sınıflar, oyunda karşılığı olan parçaları temsil eder. Bir bölümün verisi, board'un hücreleri, taşın hareketi ve oyuncunun kalan hamleleri kendi sorumlulukları içinde ele alınır.

Normalde ana menüden bir bölüm açar, taşları eşleştirir ve hamlelerin bitmeden hedefleri tamamlamaya çalışırsın. Kod da aynı akışı takip eder: bölüm yüklenir, board kurulur, takaslar çözülür ve sonuç oyun oturumuna işlenir.

Bölüm
------

Bölüm, oynayacağın board'un bütün ayarlarını tutar. Hamle sayısı, puan ve potion hedefleri, kapalı hücreler ve kullanabileceğin special strike sayıları bir `LevelData` asset'i üzerinden belirlenir.

`LevelCatalog`, bütün levelleri oynanış sırasına göre tutar. `LevelLoader` ise menüde seçilen level'ı `GameBoard` sahnesine taşır. Yeni bir bölüm eklemek için yeni bir `LevelData` oluşturup kataloğa eklemen yeterlidir.

Yeni bir bölüm hazırlamak için:

1. **Assets → Create → Scriptable Objects → LevelData** üzerinden yeni bir asset oluştur.
2. Hamle sayısını, puan hedefini, potion hedeflerini ve special strike sayılarını gir.
3. `arrayLayout` üzerinden kapatmak istediğin hücreleri işaretle. **İşaretli hücre kapalıdır.** Inspector'daki görünür 8 satır board'u temsil eder; en alttaki satır board'un alt satırıdır.
4. Hazırladığın asset'i `LevelCatalog` listesine oynanış sırasıyla ekle.

Kullanım:

```csharp
// Ana menüde Play: oyuncunun geçtiği son bölümden sonrakini seç
LevelData level = catalog.GetPlayableLevel(user.highestCompletedLevel);

LevelLoader.selectedLevel = level;
SceneManager.LoadScene(ButtonControl.GameBoardScene);

// Bölüm kazanılınca: sıradakine geç (son bölümse null döner)
LevelData next = catalog.GetNext(level);
```

Board
------

Board, potion'ların swap edildiği, match'lerin bulunduğu ve cascade'lerin çözüldüğü ana oyun alanıdır. Bu akışı `PotionBoard` yönetir: swap'i alır, match'leri buldurur, potion'ları temizletir ve boşalan hücreleri yeniden doldurur.

`PotionBoard` bütün bu işleri tek başına yapmaz. Aynı GameObject üzerindeki component'ler akışın farklı bölümlerini yönetir:

| Bileşen | Görevi |
|---|---|
| `BoardRefill` | İlk dolumu, object pool'u ve boş hücrelerin doldurulmasını yönetir |
| `SpecialChain` | Bomb ve rocket zincirlerini ve özel taş kombolarını yönetir |
| `StrikePresentation` | Special strike animasyonlarını oynatır |
| `BoardInput` | Oyuncunun dokunuşunu tap veya swap işlemine çevirir |
| `BoardEffects` | Particle effect'leri ve sesleri yönetir |
| `BoardMask` | Level verisindeki board şekline göre maske üretir |

`PotionBoard` bu component'leri çağırır, fakat component'ler `PotionBoard`'u geri çağırmaz. Böylece ana oyun akışının kontrolü tek bir yerde kalır.

Kullanım:

```csharp
// BoardInput dokunuşları board'a iletir; kabul kuralları board'dadır
if (board.AcceptsInput)
{
    board.TrySwap(first, second);   // komşu, duran ve kendi hücresindeki iki taş; değilse false
    board.TryTap(potion);           // özel taşsa patlatır, güçlendirici seçiliyse ona iletir
}

// Bir UI paneli açıkken board dokunuş almaz
board.InputLocked = true;
```

Grid
------

Board üzerindeki her potion bir hücreye aittir. `BoardGrid`, hücrelerin ve hücrelerde bulunan potion'ların kaydını tutar. Bir potion'ın yeri değiştiğinde hem grid üzerindeki hücresi hem de potion'ın koordinatları birlikte güncellenir. Böylece diğer sınıfların hücre dizisini doğrudan değiştirmesine gerek kalmaz.

Boyutlar `BoardDefinition` içinde tanımlanır: `VisibleWidth` 6, `VisibleHeight` 8, `SpawnHeight` 7.

Kullanım:

```csharp
BoardGrid grid = new BoardGrid(level.arrayLayout);    // layout'ta true = kapalı hücre

Potion potion = grid.PotionAt(new Vector2Int(2, 0));  // kapalı, boş ya da board dışıysa null
grid.Swap(first, second);                             // hücreler ve koordinatlar birlikte değişir

bool settled = !grid.AnyPotionMoving();
```

Match
------

Yan yana gelen üç aynı potion normal bir match oluşturur. Düz bir hatta dört veya daha fazla potion match olduğunda rocket, T veya L şeklinde bir match oluştuğunda ise bomb oluşturulur. Bu grupları `MatchFinder` bulur. Her çağrıda grid'in o anki durumunu okur ve önceki aramanın sonucunu saklamaz.

Match grubunda special potion'a dönüşecek taş `ProtectedPotion` olarak tutulur. Match bir swap sonucunda oluştuysa swap'i yapan potion korunur; cascade sırasında oluştuysa gruptan rastgele bir potion seçilir.

Swap'ten sonra ilk olarak yer değiştiren iki potion'ın çevresi kontrol edilir. Board'un geri kalanındaki match'ler ise bütün potion'lar durduktan sonra cascade sırasında bulunur.

Kullanım:

```csharp
MatchFinder finder = new MatchFinder(grid);

List<MatchResult> groups = finder.FindAround(first, second);  // takastan sonra
groups = finder.FindAll();                                     // board durunca

foreach (MatchResult group in groups)
{
    // Uzun hat → roket, T/L → bomba
    if (group.IsSuperMatch) Debug.Log($"{group.Direction}: {group.ProtectedPotion.name}");
}
```

Potion
------

`Potion`, board'da gördüğün taşların ana component'idir. Potion type'ını, bulunduğu hücreyi ve hareket durumunu tutar. Bir rocket veya bomb'a dönüştüğünde kendi görselini de değiştirir.

Potion'ın konumu ve oyun logic'i ana objede, sprite ve animasyonu ise child objede bulunur. Bu ayrım sayesinde kodla verilen düşüş esnemesi ve kırılma küçülmesi, Animator ile aynı anda sorunsuz şekilde uygulanabilir.

Bir potion kırıldığında GameObject'i silinmez. Object pool'a gönderilir ve board tekrar doldurulurken yeniden kullanılır.

Kullanım:

```csharp
PotionType type = potion.PotionType;   // Red, Blue, Yellow, Green, Bomb, Rocket

potion.BecomeRocket(vertical: true);   // dikey roket sütun temizler
potion.BecomeBomb();

potion.MoveToTarget(cellCenter);       // takas: arkasında duman izi bırakır
potion.MoveToDown(cellCenter);         // düşüş: yerçekimiyle hızlanır, inince esner

if (!potion.IsMoving) { /* hareket ve iniş animasyonu bitti */ }
```

Cascade ve Special Chain
------

Cascade, potion'lar temizlendikten sonra yukarıdaki potion'ların düşmesi ve bu düşüş sonucunda yeni match'lerin oluşmasıdır. Yeni match kalmayana kadar temizleme, refill ve match kontrolü devam eder.

Special chain ise bir rocket'ın bomb'a temas edip onu patlatması gibi özel taşların birbirini tetiklemesidir. Her patlama kendi coroutine'i üzerinden ilerlediği için efektler sırasıyla oynarken oyun akışı da devam eder.

`ChainContext`, hangi hücrelerin tetiklendiğini ve kaç patlamanın hâlâ devam ettiğini tutar. Aynı hücre bir chain içinde iki kez tetiklenmez. Son patlama tamamlandığında chain biter ve board refill edilir.

İki special potion birleştirildiğinde de aynı sistem kullanılır:

- **Bomb + Bomb:** Birleşme animasyonundan sonra 7 × 7 alanı merkezden dışa doğru halka halka temizler.
- **Rocket + Rocket:** Artı şeklinde patlar; bir rocket satırı, diğeri sütunu aynı anda temizler.

Kullanım:

```csharp
// PotionBoard'da bir özel taşa dokunulduğunda
private IEnumerator TapDetonate(Potion special)
{
    yield return specialChain.ExplodeChain(special);   // zincirin tüm halkaları bitene kadar
    yield return RefillAndCascade();                   // boşlukları doldur, yeni eşleşmeleri çöz

    GameManager.Instance.ProcessTurn();                // bir hamle düş
}
```

Special Strikes
------

Special strike kullanmak için alt bardan bir strike seçip board üzerindeki hedefe dokunursun. Her level'ın verdiği kullanım hakları ayrıdır ve special strike kullanmak normal hamle harcamaz.

- **Hammer:** Butondan hedefe uçar; merkezdeki potion'ı ve dört komşusunu kırar.
- **Bomb:** Hedefe fırlatılır ve 3 × 3 alanı temizler.
- **Cannon:** Board bir hücre kenara kayar; oyuncu bir satır seçer ve cannonball o satırı baştan sona temizler.

`SpecialStrikes`, hangi strike'ın seçildiğini, kaç kullanım hakkı kaldığını ve hangi hücrelerin etkileneceğini belirler. Hücreleri temizleme işini board yapar. Etki alanında bomb veya rocket varsa mevcut special chain sistemi üzerinden onlar da tetiklenir.

Kullanım:

```csharp
// Hammer: hücre listesini SpecialStrikes hesaplar (merkez + dört komşu)
board.TryRunStrike(StrikeKind.Hammer, origin, hammerCells, hammerButton.transform);

// Bomb: 3x3 alanı board kendisi temizler
board.TryRunStrike(StrikeKind.Bomb, origin, null, bombButton.transform);

// Cannon: önce board kayar, oyuncu bir satıra dokununca ateşlenir
board.TryBeginCannonAim();
board.TryRunStrike(StrikeKind.Cannon, origin, null);

// false dönerse vuruş başlamadı: hak düşmez, seçim açık kalır
```

Game Session
------

Board potion'larla ilgilenirken `GameSession` level'ın ilerleyişini takip eder. Puan, kalan hamleler, potion hedefleri ve bölüm sonucu burada tutulur. `GameSession`, Unity'den bağımsız saf bir C# sınıfıdır.

Son hedefi swap, cascade, special potion veya special strike ile tamamlayabilirsin. Bütün hedefler tamamlandığında level kazanılır; hedefler tamamlanmadan hamleler biterse kaybedilir.

`GameManager`, `GameSession` içindeki sonucu Unity sahnesine aktarır. HUD, karakter animasyonları, sesler, konfeti ve oyun sonu panelleri bu sınıf üzerinden güncellenir.

Kullanım:

```csharp
GameSession session = new GameSession(level);   // hedefler kopyalanır, asset değişmez

session.AddPoints(10);
session.RegisterCleared(PotionType.Red);        // kırmızı toplama hedefini bir düşürür
session.EndTurn();                              // hamle 0 olursa sonuç Lost olur

if (session.Outcome == SessionOutcome.Won) { /* tüm hedefler tamam */ }
```

Oyuncu ve Firebase
------

Oyuncu, Firebase Authentication üzerinde oluşturulan anonim hesapla temsil edilir. `FirebaseBootstrap` oyun açıldığında Firebase'i hazırlar, auth işlemini tamamlar ve `users/{uid}` dökümanını Firestore'dan yükler. Oyuncu ilk kez giriş yapıyorsa yeni bir kullanıcı dökümanı oluşturur.

`FirebaseBootstrap`, `DontDestroyOnLoad` ile sahne değişimlerinde korunur. Can, altın, level ilerlemesi ve profil güncellemeleri bu servis üzerinden yönetilir. Kullanıcı verisi değiştiğinde `UserReady` event'i yayınlanır; açık UI component'leri bu event'i dinleyerek kendini günceller.

Kullanım:

```csharp
// Kullanıcı verisi her değiştiğinde (giriş, can, altın, profil) tetiklenir
FirebaseBootstrap.UserReady += OnUserChanged;

FirebaseBootstrap player = FirebaseBootstrap.Instance;

player.CompleteLevel(level.level, points, goldReward: 50);   // ilerleme, altın, clan puanı: tek batch
player.SpendLife();                                           // bölüm kaybedilince ya da bırakılınca
player.RegenerateLives();                                     // süresi dolan canları ekler
```

Clan
------

Clan, oyuncuların bir araya geldiği takım sistemidir. Altın harcayarak kendi clan'ını kurabilir veya arama bölümünden uygun bir clan'a katılabilirsin. Clan leader'ı ayrılırsa liderlik en yüksek level'a sahip üyeye devredilir. Son üye de ayrıldığında clan silinir.

Clan chat gerçek zamanlı çalışır. Can istekleri de aynı chat içinde özel bir mesaj tipi olarak gösterilir. Böylece normal mesajlar ve can istekleri için ayrı sistemler yerine tek bir liste ve tek bir Firestore listener kullanılır.

Bu işlemler iki static servis üzerinden yönetilir: `ClanService` clan üyeliğini ve ayarlarını, `ClanChatService` ise chat ve can isteklerini yönetir.

Kullanım:

```csharp
ClanService.CreateClan(new ClanData { name = "Kara Bayrak", maxMembers = 30 }, (ok, message) => { });
ClanService.SearchClans("kara", 30, clans => resultList.Show(clans));
ClanService.JoinClan(clan, (ok, message) => { });
ClanService.LeaveClan((ok, message) => { });

ListenerRegistration chat = ClanChatService.Listen(clanId, 50, Render);
ClanChatService.SendChat("Selam!");
ClanChatService.SendLifeRequest();
ClanChatService.DonateLife(request);                  // bağışçı yalnızca kendi kimliğini ekler
ClanChatService.ClaimLives(request, gained => { });   // canları isteği atan toplar
chat.Stop();
```

------
Başlarken
======
Gereksinimler
------

- Unity Hub ve **Unity 6000.5.0f1**; iOS ve/veya Android build support modülleriyle birlikte
- Node.js (yalnızca Firestore kurallarını yayınlamak için)

Kurulum
------

**1. Klonla ve aç.**

```bash
git clone git@github.com:mahmut483/match3.git
```

İndirdiğin proje klasörünü Unity Hub üzerinden aç ve Unity'nin asset import işlemini tamamlamasını bekle.

**2. Firebase masaüstü kütüphanelerini ekle.**

`Assets/Firebase/Plugins/x86_64/` klasörü GitHub'ın dosya boyutu sınırı nedeniyle repoya dahil edilmedi. Firebase'in Unity Editor üzerinde çalışması için [Firebase Unity SDK 13.15.0](https://firebase.google.com/docs/unity/setup) içindeki `FirebaseAuth.unitypackage` ve `FirebaseFirestore.unitypackage` paketlerini projeye import et.

**3. Firebase projesini bağla.**

Repo, `match3-3dc9b` Firebase projesine göre yapılandırılmış durumda. Kendi Firebase projenle çalışmak için:

1. Firebase Console üzerinden iOS (`com.MahmutCompany.match3`) ve Android uygulamalarını ekle.
2. `GoogleService-Info.plist` ve `google-services.json` dosyalarını `Assets/` klasörüne koy.
3. **Authentication** bölümünde **Anonymous** sign-in yöntemini aç ve bir **Firestore** database oluştur.
4. Kuralları repo kökünden yayınla:
   ```bash
   npx firebase-tools login
   npx firebase-tools deploy --only firestore:rules
   ```
5. **Firestore → Time-to-live** bölümünde `messages` collection group'u ve `expireAt` alanı için bir TTL policy ekle.

Firebase kurulumu hakkında ek bilgi için [`docs/firebase.md`](docs/firebase.md) dosyasına bakabilirsin.

Çalıştırma
------

`Assets/Scenes/MainMenu.unity` sahnesini açıp Unity Editor'da **Play**'e bas. Anonymous sign-in tamamlanıp oyuncu verisi Firestore'dan yüklendiğinde menüdeki Play butonu aktif hale gelir.

Board'u doğrudan test etmek için `Assets/Scenes/GameBoard.unity` sahnesini açabilirsin. Bu durumda **GameManager → Level Data** alanında bulunan test level'ı yüklenir. Can ve level ilerlemesi gibi Firebase'e bağlı özellikler bu kullanımda devre dışı kalır.

------
Proje hakkında
======
Tasarım dokümanları
------

- [Bölüm sistemi](docs/superpowers/specs/2026-08-18-level-system-design.md)
- [Firestore backend](docs/superpowers/specs/2026-08-19-firestore-backend-design.md)
- [Özel vuruşlar (güçlendiriciler)](docs/superpowers/specs/2026-09-08-special-strikes-design.md)
- [GameBoard ayrıştırma refactor'ü](docs/superpowers/specs/2026-09-21-gameboard-split-design.md)

Bilinen sınırlamalar
------

- **Oyun sonuçları server tarafında doğrulanmıyor.** Firestore security rule'ları veri biçimini ve erişimi kontrol ediyor, fakat oynanan hamlelerin geçerli olup olmadığını doğrulamıyor. Bu nedenle değiştirilmiş bir client puan veya altın değerlerine müdahale edebilir. Gerçek para içeren özellikler eklenmeden önce ekonomi işlemlerinin Cloud Functions gibi server-side bir sisteme taşınması gerekiyor.
- **Bekleme süreleri geliştirme için kısa tutuldu:**
  - Can yenilenmesi: 60 sn (`LifeRules.RegenSeconds`)
  - Can isteği bekleme süresi: 10 sn (`ClanChatService.RequestCooldownSeconds`)
- **Android** için gereken `google-services.json` repoda yok.
- Projede henüz automated test bulunmuyor.

Third-party asset'ler
------

Projede kullanılan third-party package ve asset'ler aşağıda listeleniyor. Her biri kendi lisans koşullarına tabidir.

- [Spine Runtimes](https://esotericsoftware.com/spine-runtimes) (spine-unity), Esoteric Software
- Cartoon FX Remaster (Free), Jean Moreno (JMO)
- Gem Hunter Match örnek asset'leri, Unity Technologies
- Hyper Casual FX, Lana Studio
- Farm Game UI – Simple 2D UI, maanetorn
- 2D Casual UI
- Firebase Unity SDK ve External Dependency Manager, Google
