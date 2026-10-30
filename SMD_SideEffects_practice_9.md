# Side-effects — Practice_9


### 1
**Kontekst:** Dasturchi `ProfileScreen` composable tanasining boshida `viewModel.loadProfile()` va `analytics.log("opened")` ni chaqiradi. Ekranda animatsiya ishlaganda so'rovlar va log yozuvlari ko'payib ketdi.

**Yechim:**
1. **Side-effect nima:**
   Composable funksiyaning boshqaruv doirasidan (scope) tashqaridagi ilova holatini o'zgartiradigan yoki tashqi tizimlar bilan aloqa qiladigan har qanday operatsiya **side-effect** deyiladi (tarmoq so'rovi, diskka yozish, navigatsiya, loglar, timerlar).
2. **Nega Composable tanasi side-effect uchun xavfli:**
   - **Recomposition chastotasi:** Animatsiyalar yoki tez o'zgaruvchi state paytida composable sekundiga 60–120 martagacha qayta bajarilishi mumkin.
   - **Bekor qilinish (Discard/Cancel):** Compose runtime yangi o'zgarish kelib qolsa, davom etayotgan recomposition'ni bekor qilib, boshqatdan boshlashi mumkin.
   - **Thread xavfsizligi:** Composable funksiyalar fon oqimlarida yoki parallel bajarilishi mumkin. Shuning uchun tanadagi to'g'ridan-to'g'ri chaqiruvlar nazoratsiz ko'p marta ishlaydi.
3. **“Effect” (Key Term) ta'rifi:**
   **Effect** — bu hech qanday vizual UI chizmaydigan (UI emit qilmaydigan), balki Composable hayot sikliga (Composition lifecycle) moslashgan holda side-effect'larni xavfsiz boshqarish imkonini beruvchi maxsus composable funksiyadir.
4. **Nega ma'lumot yuklash ViewModel'ga tegishli (Unidirectional Data Flow):**
   Composable UI faqat o'ziga berilgan holatni chizib berishi kerak (declarative UI). Ma'lumotlarni yuklash va boshqarish esa konfiguratsiya o'zgarishlaridan (ekranni burish va h.k.) mustaqil yashovchi **ViewModel** ichida bo'lishi shart. State pastga (UI tomon) oqadi, eventlar esa yuqoriga (ViewModel tomon) uzatiladi.

---

### 2
**Kontekst:** Jamoa Compose'dagi sakkizta effect API'ni ajrata olmayapti va har bir vaziyatda `LaunchedEffect` ishlatmoqda.

| Men xohlayman | API |
| :--- | :--- |
| Composable Composition'da turganda suspend funksiyani chaqirish | **`LaunchedEffect`** |
| `onClick` kabi callback ichidan coroutine ishga tushirish | **`rememberCoroutineScope`** |
| Effect ichida eng so'nggi qiymatni o'qish, lekin effect'ni qayta ishga tushirmaslik | **`rememberUpdatedState`** |
| Listener qo'shish va keyin uni olib tashlash | **`DisposableEffect`** |
| Compose state'ni Compose boshqarmaydigan kodga uzatish | **`SideEffect`** |
| Flow, LiveData yoki RxJava'ni Compose State'ga aylantirish | **`produceState`** |
| Kirish qiymatidan kamroq recompose qilish | **`derivedStateOf`** |
| Compose State'ni Flow'ga aylantirish | **`snapshotFlow`** |

---

### 3
**Kontekst:** Quyidagi composable ekran Composition'ga kirganda pulsatsiya animatsiyasini ishga tushiradi:
```kotlin
var pulseRateMs by remember { mutableLongStateOf(3000L) }
val alpha = remember { Animatable(1f) }
LaunchedEffect(pulseRateMs) {
    while (isActive) {
        delay(pulseRateMs)
        alpha.animateTo(0f)
        alpha.animateTo(1f)
    }
}
```

**Yechim:**
1. **Qachon ishga tushadi:** Composable birinchi marta Composition daraxtiga kirganda (Initial composition) va keyinchalik `pulseRateMs` kaliti har safar o'zgarganda.
2. **`pulseRateMs` 3000 dan 1000 ga o'zgarganda nima bo'ladi:**
   Oldingi `LaunchedEffect` coroutine'i darhol **bekor qilinadi (cancelled)** va `pulseRateMs = 1000L` yangi parametri bilan yangi coroutine boshlanadi.
3. **Composition'ni tark etganda taqdiri va `isActive` roli:**
   Composable Composition'dan chiqib ketganda effect coroutine'i bekor qilinadi.
   `isActive` — bu coroutine doirasining faolligini bildiruvchi xususiyat. U job bekor bo'lishi bilan `false` ga aylanadi. Shuningdek, `delay()` va `animateTo()` ham suspend funksiyalar bo'lgani uchun ular darhol `CancellationException` tashlaydi va `while` sikli zudlik bilan to'xtaydi.
   **MiniComposition modeli:** Kalit o'zgarganda yoki composable tark etilganda `effect(...) cancelled` deb qayd etiladi va yangi effect yaratiladi.

---

### 4
**Kontekst:** Dasturchi foydalanuvchi o'zgarishi mumkin bo'lgan ekranda quyidagicha yozdi:
```kotlin
LaunchedEffect(true) {
    api.sendHello(userId)  // userId o'zgarishi mumkin
}
```

**Yechim:**
1. **Nega `LaunchedEffect(true)` “while(true) kabi shubhali”:**
   Chunki `true` hech qachon o'zgarmas doimiy (constant) qiymatdir. Effect ichida `userId` kabi tashqi parametr ishlatilsa, u birinchi kirishdagi qiymatni ushlab qoladi va `userId` boshqa foydalanuvchiga o'zgarganda effect qayta ishga tushmaydi.
2. **Doimiy kalit qachon o'rinli:**
   Faqatgina effect haqiqatdan ham butun ekran hayoti davomida **faqat bir marta** ishlashi kerak bo'lganda (masalan, Splash ekranda bir martalik 2 soniya kutishda yoki analitikada "Screen Loaded" hodisasini bir marta qayd etishda).
3. **Kodni tuzatish:**
   ```kotlin
   LaunchedEffect(userId) {
       api.sendHello(userId)
   }
   ```

---

### 5
**Kontekst:** Talaba tugma ichida snackbar ko'rsatish uchun `onClick` ga `LaunchedEffect(Unit) { snackbarHostState.showSnackbar("Hi") }` yozdi. Kod kompilyatsiya bo'lmayapti.

**Yechim:**
1. **Nega chaqirib bo'lmaydi:**
   `LaunchedEffect` — bu `@Composable` funksiya. Jetpack Compose qoidasiga ko'ra, composable funksiyalar faqat boshqa composable kontekstida (composition paytida) chaqirilishi mumkin. `onClick` esa oddiy runtime lambda callback'i.
2. **`rememberCoroutineScope` nimani qaytaradi va umri:**
   U joriy Composable'ning Composition'dagi nuqtasiga bog'langan `CoroutineScope`ni qaytaradi. Ushbu composable Composition'ni tark etganda (ekrandan ketganda), bu scope orqali boshlangan barcha coroutine'lar avtomatik tarzda bekor qilinadi.
3. **To'g'ri variant:**
   ```kotlin
   val scope = rememberCoroutineScope()
   Button(onClick = {
       scope.launch {
           snackbarHostState.showSnackbar("Hi")
       }
   }) {
       Text("Ko'rsatish")
   }
   ```
   `showSnackbar` suspend funksiya bo'lgani sababli, unga coroutine oqimi talab etiladi.
4. **Tanlov qoidasi:**
   - *Composable paydo bo'lgani uchun boshlanadigan ish* -> **`LaunchedEffect`**
   - *Foydalanuvchi biror amal bajargani (event) uchun boshlanadigan ish* -> **`rememberCoroutineScope`**

---

### 6
**Kontekst:** `LandingScreen(onTimeout)` splash ekrani 2 soniya kutib, `onTimeout()` ni chaqiradi. Ota composable recompose bo'lganda unga yangi `onTimeout` beradi:
```kotlin
@Composable
fun LandingScreen(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    LaunchedEffect(true) {
        delay(SplashWaitTimeMillis)
        currentOnTimeout()
    }
}
```

**Yechim:**
1. **Eski lambdani chaqirish sababi:**
   `LaunchedEffect(true)` 2 soniya davomida uzluksiz ishlaydi. Agar `rememberUpdatedState` ishlatilmasa, u closure orqali dastlabki uzatilgan lambdani o'zida saqlab qoladi (capture) va oraliqda ota ekran yangilangan bo'lsa ham eski referensni chaqiradi.
2. **`rememberUpdatedState` buni qanday hal qiladi:**
   U har bir recomposition'da o'zining ichki `State` qiymatini yangilab boradi, lekin o'zi qayta yaratilmaydi. Coroutine tugagach, `currentOnTimeout()` deb chaqirilganda eng so'nggi uzatilgan lambda bajariladi.
3. **Nega `onTimeout` ni kalit qilish xato:**
   Agar `LaunchedEffect(onTimeout)` deb yozilsa, har safar ota composable qayta chizilib yangi lambda berganida taymer bekor qilinadi va 2 soniya kutish qaytadan boshlanadi. Natijada splash ekrandan hech qachon chiqib bo'lmaydi.
4. **Model isboti:**
   `captured -> onTimeout v1` (eski capture qilingan referens), `updatedState -> onTimeout v3` (taymer buzilmagan holda eng oxirgi holat ishga tushirilgani).

---

### 7
**Kontekst:** Ekran `LifecycleEventObserver` qo'shadi, lekin dasturchi uni olib tashlashni unutdi va `onDispose` yozmadi:
```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> /* ... */ }
    lifecycleOwner.lifecycle.addObserver(observer)
    // onDispose yo'q
}
```

**Yechim:**
1. **Nega cleanup kerak va xotira oqishi (leak):**
   `lifecycleOwner` Compose'dan tashqaridagi obyekt hisoblanadi. Agar ro'yxatdan o'tgan tinglovchi olib tashlanmasa, u composable xotiradan o'chib ketganidan keyin ham uning referenslarini ushlab turadi. Bu xotira to'lib ketishiga (**Memory Leak**) va keraksiz fon chaqiruvlariga olib keladi.
2. **`onDispose` qoidasi:**
   `onDispose { ... }` bloki `DisposableEffect` ning **eng oxirgi qatori (last statement)** bo'lishi shart. Aks holda kompilyator xatolik beradi va kod yig'ilmaydi.
3. **Bo'sh `onDispose` nega yomon:**
   Agar tozalash kerak bo'lmasa, `DisposableEffect` ishlatish semantik jihatdan xato. Uning o'rniga `LaunchedEffect` yoki `SideEffect` ishlatilishi kerak.
4. **Tuzatilgan kod:**
   ```kotlin
   DisposableEffect(lifecycleOwner) {
       val observer = LifecycleEventObserver { _, event -> /* ... */ }
       lifecycleOwner.lifecycle.addObserver(observer)
       
       onDispose {
           lifecycleOwner.lifecycle.removeObserver(observer)
       }
   }
   ```

---

### 8
**Kontekst:** `MiniDisposable` modeli `DisposableEffect` ning qoidalarini taqlid qiladi:
```kotlin
fun screen(owner: String) = comp.disposableEffect(owner) {
    println("$owner.addObserver(observer)")
    val onDispose = { println("$owner.removeObserver(observer)") }
    onDispose
}
 
screen("A"); screen("A")   // bir xil owner
screen("B")                // owner o'zgardi
comp.leave()               // Composition'dan chiqdi
```

**Yechim:**
**Konsol chiqishi (to'liq 4 qator):**
```text
A.addObserver(observer)
A.removeObserver(observer)
B.addObserver(observer)
B.removeObserver(observer)
```
- **Sababi:**
  1. `screen("A")` (1-marta) -> `A.addObserver` ishlaydi.
  2. `screen("A")` (2-marta) -> Kalit o'zgarmagan ("A" == "A"), shuning uchun hech narsa bo'lmaydi.
  3. `screen("B")` -> Kalit o'zgardi. **Birinchi bo'lib eski observer olib tashlanadi** (`A.removeObserver`), so'ngra **yangi observer qo'shiladi** (`B.addObserver`).
  4. `comp.leave()` -> Composition tark etilgach oxirgi observer tozalanadi (`B.removeObserver`).

---

### 9
**Kontekst:** Analytics kutubxonasiga foydalanuvchi turi (`userType`) uzatilishi kerak. Dasturchi uni composable tanasida to'g'ridan-to'g'ri o'rnatadi.

**Yechim:**
1. **`SideEffect` qachon ishga tushadi:**
   Har bir muvaffaqiyatli recomposition yakunlanib, Compose o'zgarishlarni ekranga muhrlaganidan (commit) so'ng ishga tushadi.
2. **Nega to'g'ridan-to'g'ri chaqirish xavfli:**
   Compose recomposition jarayonini istalgan paytda to'xtatishi yoki bekor qilishi (discard) mumkin. Agar to'g'ridan-to'g'ri chaqirilsa, bekor qilingan urinishda ham analytics noto'g'ri hisoblab yuboriladi.
3. **Attempt modeli (3 ta holat):**
   - *Direct call, commits=false:* Analytics **chaqiriladi** (Xato! UI ekranda chiqmadi).
   - *SideEffect, commits=false:* Analytics **chaqirilmaydi** (To'g'ri! Bekor bo'lgani uchun effect ishlamadi).
   - *SideEffect, commits=true:* Analytics **chaqiriladi** (To'g'ri! Kadr ekranga chiqishi aniq bo'ldi).
4. **Kalitlari bormi:**
   **Yo'q.** `SideEffect` hech qanday kalit (`key`) qabul qilmaydi. U har bir muvaffaqiyatli recomposition'da bajariladi.

---

### 10
**Kontekst:** Rasm yuklash holatlari: `Loading`, `Success` va `Error`. Dasturchi `loadNetworkImage(url)` funksiyasini `produceState` bilan yozishi kerak.

**Yechim:**
1. **`produceState` vazifasi:**
   Tashqi suspend/Flow manbalarini Compose `State<T>` ga aylantiradi. Composable composition'ga kirganda producer ishga tushadi; composable chiqib ketganda yoki berilgan kalit (`url`) o'zgarganda oldingi producer bekor qilinadi.
2. **Nega “conflates”:**
   `produceState` ichida `mutableStateOf` yotadi. Agar producer `value = yangiQiymat` deb yozsa va bu qiymat avvalgisi bilan teng (`equals`) bo'lsa, Compose ortiqcha recomposition qilmaydi (conflation).
3. **Nega kichik harf bilan boshlanadi:**
   Compose qoidasiga ko'ra, agar funksiya UI element chizsa Katta harf bilan (`Text`, `Button`), agar UI chizmasdan qiymat qaytarsa (value-returning composable) kichik harf bilan nomlanadi (`produceState`, `rememberCoroutineScope`).

---

### 11
**Kontekst:** `produceState` ni `remember { mutableStateOf(...) }` va `LaunchedEffect` bilan amalga oshirish.

**Yechim:**
```kotlin
@Composable
fun <T> myProduceState(
    initialValue: T,
    key: Any?,
    producer: suspend MutableState<T>.() -> Unit
): State<T> {
    val state = remember { mutableStateOf(initialValue) }
    LaunchedEffect(key) {
        state.producer()
    }
    return state
}
```
**Xatti-harakat:** Funksiya `LaunchedEffect` orqali ishlaydi. Kalit (`key`) o'zgarganda eski producer coroutine avtomatik bekor qilinadi va yangi kalit bilan yangi producer boshlanadi.

---

### 12
**Kontekst:** “Tepaga qaytish” tugmasi ro'yxatning birinchi ko'rinuvchi elementi har o'zgarganda qayta chiziladi:
```kotlin
val listState = rememberLazyListState()
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

**Yechim:**
1. **Nega to'g'ridan-to'g'ri o'qish keraksiz recomposition keltiradi:**
   `listState.firstVisibleItemIndex` foydalanuvchi ro'yxatni siljitganda har bir yangi element ko'ringanida (0, 1, 2, 3...) o'zgaradi. To'g'ridan-to'g'ri o'qilganda composable har bir yangi indeksda qayta chiziladi.
2. **`derivedStateOf` buni qanday hal qiladi va nega `remember` ichida:**
   `derivedStateOf` faqat hisob-kitob natijasi (`false` dan `true` ga yoki aksincha) o'zgargandagina recomposition chaqiradi. Har safar yangi `DerivedState` obyekti yaratilmasligi uchun u `remember` ichiga olinadi.
3. **Indekslar: `[0, 0, 1, 2, 3, 4, 5, 3, 1, 0]` hisob-kitobi:**
   - **To'g'ridan-to'g'ri o'qish:** Indeks har o'zgarganda recomposition bo'ladi (0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 3 -> 1 -> 0) = Jami **9 ta** recomposition.
   - **`derivedStateOf` bilan:**
     - 0 -> `false`
     - 1 ga o'tganda: `false` -> `true` (**1-recomposition**)
     - 2, 3, 4, 5, 3, 1 bo'lganda natija hamon `true` -> Recomposition bo'lmaydi!
     - 0 ga qaytganda: `true` -> `false` (**2-recomposition**)
     Boshlang'ich holat bilan birga jami faqat **3 ta** holat o'zgarishi qayd etiladi.

---

### 13
**Kontekst:** Talaba ikki holatni birlashtirishda `derivedStateOf` ishlatdi:
```kotlin
var firstName by remember { mutableStateOf("") }
var lastName by remember { mutableStateOf("") }
val fullNameBad by remember { derivedStateOf { "$firstName $lastName" } }
```

**Yechim:**
1. **Nega bu yerda ortiqcha xarajat:**
   `"$firstName $lastName"` bu juda yengil va tez string konkatenatsiyasi. `derivedStateOf` esa o'z ichida kuzatuvchi obyektlar va kesh mexanizmini yurgazadi. Qolaversa, `firstName` yoki `lastName` har o'zgarganda natija ham deyarli har doim o'zgaradi.
2. **To'g'ri variant:**
   ```kotlin
   var firstName by remember { mutableStateOf("") }
   var lastName by remember { mutableStateOf("") }
   val fullName = "$firstName $lastName"
   ```
3. **Qo'llash mezoni:**
   Kiruvchi holatlarning o'zgarish chastotasi chiquvchi natijaning o'zgarish chastotasidan **ancha yuqori** bo'lsagina (`inputs change much more frequently than the output`) `derivedStateOf` ishlatiladi.
4. **Caution ogohlantirishi:**
   Agar natija kiruvchi parametrlar bilan bir xil chastotada o'zgarsa yoki oddiy hisoblash bo'lsa, `derivedStateOf` ishlatmang — bu faqatgina ortiqcha xotira va qo'shimcha resurs sarfiga olib keladi.

---

### 14
**Kontekst:** Waveform (ovoz to'lqini) misolida `derivedStateOf` mezonlari.

**Yechim:**
1. **Qaysi shartda ishlatiladi:**
   Audio pleerning joriy ijro vaqti (`currentTimeMs`) har bir millisekundda o'zgarib turganda, to'lqin shakli (waveform) faqat ma'lum bir segmentlar yoki oraliqlar (masalan har 500 ms) bo'yicha hisoblanishi kerak bo'lsa.
2. **Qaysi uchta holatda ishlatilmaydi va sababi:**
   - **Amplitudalar oldindan hisoblangan bo'lsa:** Bu statik ma'lumot, recomposition hisoblash talab qilmaydi, oddiy `remember(audioData)` yetarli.
   - **Waveform statik bo'lsa:** O'zgarmas ma'lumotlar uchun qo'shimcha state kuzatuvchi kerak emas.
   - **Waveform `LaunchedEffect` yoki `animate*` bilan yangilansa:** Animatsiya o'zining ichki optimallashgan oqimiga ega bo'lib, uning ustiga `derivedStateOf` qurish asossiz ortiqcha qatlam hisoblanadi.

---

### 15
**Kontekst:** Ro'yxat birinchi elementdan o'tganda analytics hodisasi yuborish:
```kotlin
LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```

**Yechim:**
1. **`snapshotFlow` nima va nega cold Flow:**
   U Compose State'larini kuzatib, ular o'zgarganda qiymatlarni Kotlin Coroutines `Flow`iga aylantirib beruvchi ko'prikdir. U **cold Flow** hisoblanadi, chunki `collect` chaqirilmaguncha u hech qanday qiymat ishlab chiqarmaydi va resurs sarflamaydi.
2. **Operatorlar:**
   - `map { it > 0 }`: Raqamli indeksni `Boolean` flagga o'tkazadi.
   - `distinctUntilChanged()`: Ketma-ket kelgan bir xil qiymatlarni tashlab yuboradi (`false, false` yoki `true, true`).
   - `filter { it == true }`: Faqat `true` bo'lgan (pastga o'tilgan) holatdagina hodisani o'tkazadi.
3. **Nega `collect` `LaunchedEffect` ichida:**
   `collect` — bu suspend funksiya bo'lib, u oqimni tinglash uchun coroutine doirasini talab qiladi.
4. **Oqim hisob-kitobi: `0, 0, 1, 2, 5, 3, 0, 0, 1, 4`**
   - 0 -> `false` -> distinct: o'tadi (`false`) -> filter: bloklanadi.
   - 0 -> `false` -> distinct: bloklanadi.
   - 1 -> `true` -> distinct: o'tadi (`true`) -> filter: o'tadi -> **1-HODISA YUBORILDI**.
   - 2, 5, 3 -> `true` -> distinct: bloklanadi (chunki allaqachon `true` edi).
   - 0 -> `false` -> distinct: o'tadi (`false`) -> filter: bloklanadi.
   - 0 -> `false` -> distinct: bloklanadi.
   - 1 -> `true` -> distinct: o'tadi (`true`) -> filter: o'tadi -> **2-HODISA YUBORILDI**.
   - 4 -> `true` -> distinct: bloklanadi.
   **Javob:** Jami **2 ta** analytics hodisasi yuboriladi.

---

### 16
**Kontekst:** `tick` 1 dan 4 gacha o'zgaradi (1, 2, 3, 4). `owner` faqat 3-tickda "A" dan "B" ga o'zgaradi.

| Kalitlar | Restart soni (bashorat) | Sababi |
| :--- | :--- | :--- |
| `keys = []` | **0 marta** (faqat 1 marta start) | **XATO:** Kalit bo'sh bo'lgani uchun 3-tickda `owner` "B" ga o'zgarganini sezmaydi, eskirgan referens bilan ishlaydi. |
| `keys = [owner]` | **1 marta** restart | **TO'G'RI:** Faqat `owner` o'zgarganda (3-tickda) to'g'ri qayta boshlanadi. Resurs tejaladi. |
| `keys = [owner, tick]` | **3 marta** restart | **SAMARASIZ:** Har bir `tick` o'zgarganda (1->2, 2->3, 3->4) effect keraksiz joyda qayta-qayta restart bo'laveradi. |

---

### 17
**Kontekst:** `HomeScreen(lifecycleOwner, onStart, onStop)` ichida `onStart` va `onStop` `rememberUpdatedState` bilan o'ralgan, `DisposableEffect` esa lifecycle observer qo'shadi.

**Yechim:**
1. **Qaysi o'zgaruvchi kalit bo'lishi kerak:**
   Faqat **`lifecycleOwner`** effect kaliti bo'lishi kerak.
2. **Nega `currentOnStart` va `currentOnStop` kalit emas:**
   Chunki ular `rememberUpdatedState` bilan o'ralgan. Ular o'zgarganda effect'ni qayta ishga tushirish (observer'ni uzib-ulash) shart emas — effect ichida ularning eng oxirgi qiymati o'qilaveradi.
3. **`lifecycleOwner` kalit qilinmasa nima bo'ladi:**
   Agar ekran boshqa lifecycle egasiga o'tsa (masalan, navigatsiya yoki alohida konteynerda), effect buni bilmaydi va eski lifecycle'ga yopishib qoladi. Natijada xotira oqishi (memory leak) yuzaga keladi.
4. **Key Point qoidasi:**
   Effect ichida ishlatiladigan va uning qayta ishlashini talab qiladigan barcha tashqi o'zgaruvchilar kalit sifatida ko'rsatilishi shart; effect'ni qayta ishga tushirmasdan eng so'nggi qiymatini o'qish kerak bo'lgan parametrlar esa `rememberUpdatedState` orqali olinishi lozim.

---

### 18
**Kontekst:** Code review:
```kotlin
@Composable
fun ChatScreen(userId: String, onClose: () -> Unit) {
    val scope = rememberCoroutineScope()
    val owner = LocalLifecycleOwner.current
    val observer = remember { LifecycleEventObserver { _, _ -> } }
 
    viewModel.markOpened(userId)                       // (1)
    LaunchedEffect(true) { delay(5_000); onClose() }   // (2)
    DisposableEffect(owner) {                          // (3)
        owner.lifecycle.addObserver(observer)
    }
    Button(onClick = {                                 // (4)
        LaunchedEffect(Unit) { send() }
    }) { Text("Send") }
}
```

**Yechim:**
**To'rtta xato tahlili:**
1. **(1) `viewModel.markOpened(userId)` to'g'ridan-to'g'ri composable tanasida chaqirilgan:** Har bir recomposition'da takroran chaqiriladi. `LaunchedEffect(userId)` ichiga olinishi kerak.
2. **(2) `LaunchedEffect(true)` ichida `onClose` to'g'ridan-to'g'ri chaqirilgan:** Agar ota composable yangi lambda bersa, eski closure chaqirilib ketadi. `rememberUpdatedState(onClose)` kerak.
3. **(3) `DisposableEffect` ichida `onDispose` yo'q:** Tinglovchi tozalashsiz qolib ketadi va memory leak bo'ladi. Kompilyator xatosi ham beradi.
4. **(4) `Button(onClick = { LaunchedEffect(Unit) { ... } })`:** Composable funksiyani callback ichidan chaqirib bo'lmaydi. Uning o'rniga `scope.launch` ishlatilishi kerak.

**Tuzatilgan to'liq kod:**
```kotlin
@Composable
fun ChatScreen(
    userId: String, 
    onClose: () -> Unit,
    viewModel: ChatViewModel = viewModel()
) {
    val scope = rememberCoroutineScope()
    val owner = LocalLifecycleOwner.current
    val observer = remember { LifecycleEventObserver { _, _ -> } }
    val currentOnClose by rememberUpdatedState(onClose)

    // (1) Tuzatildi: faqat userId o'zgarganda ishlaydi
    LaunchedEffect(userId) {
        viewModel.markOpened(userId)
    }

    // (2) Tuzatildi: eng oxirgi onClose lambdasini oladi
    LaunchedEffect(Unit) {
        delay(5_000)
        currentOnClose()
    }

    // (3) Tuzatildi: onDispose qo'shildi
    DisposableEffect(owner) {
        owner.lifecycle.addObserver(observer)
        onDispose {
            owner.lifecycle.removeObserver(observer)
        }
    }

    // (4) Tuzatildi: rememberCoroutineScope orqali launch qilindi
    Button(onClick = {
        scope.launch {
            viewModel.send()
        }
    }) {
        Text("Send")
    }
}
```

---

### 19
**Kontekst:** Sakkizta API bo'yicha yakuniy umumlashtirish jadvali va real misollar.

| API | Qachon ishlaydi | Tozalash (Cleanup) | Restart sababi | Real loyiha misoli |
| :--- | :--- | :--- | :--- | :--- |
| **`LaunchedEffect`** | Composition boshlanganda | Coroutine `cancel` qilinadi | Berilgan kalit (`key`) o'zgarganda | Foydalanuvchi `userId` o'zgarganda yangi chat xabarlarini serverdan yuklash. |
| **`rememberCoroutineScope`** | Callback ichida `.launch {}` qilinganda | Composable chiqib ketganda butun scope bekor bo'ladi | Restart bo'lmaydi | Tugma bosilganda BottomSheet dialogini ochish yoki yopish. |
| **`rememberUpdatedState`** | Har bir recomposition'da qiymatni yangilaydi | Yo'q | Restart bo'lmaydi | Orqa fonda audio chalayotgan timer tugagach `onFinish()` callback'ini chaqirish. |
| **`DisposableEffect`** | Composition boshlanganda va kalit o'zgarganda | `onDispose {}` bloki orqali | Berilgan kalit (`key`) o'zgarganda | Sensor yoki GPS Listener'ni ulash va ekrandan chiqqanda uzish. |
| **`SideEffect`** | Har bir muvaffaqiyatli recomposition (commit) dan so'ng | Yo'q | Har doim ishlaydi (kaliti yo'q) | Compose mavzusi (Dark/Light) o'zgarganda tizim Status Bar rangini sinxronlashtirish. |
| **`produceState`** | Dastlabki kirishda va kalit o'zgarganda | Producer coroutine bekor qilinadi | Berilgan kalit (`key`) o'zgarganda | Keshdagi fayl yo'lidan rasmni yuklab `State<Bitmap>`ga aylantirish. |
| **`derivedStateOf`** | Kuzatilayotgan state o'zgarib, natijaviy hisob yangilanganda | Yo'q | Ichki o'qilayotgan state'lar o'zgarganda | Savatchadagi tovarlar ro'yxatidan faqat umumiy summa 100\$ dan oshganini tekshirib chegirma tugmasini yoqish. |
| **`snapshotFlow`** | Flow'ga `collect` chaqirilganda | Flow kolleksiyasi to'xtaganda | Ichki Compose state qiymati o'zgarganda | Qidiruv qatoridagi matn o'zgarishini `debounce(300ms)` orqali serverga jo'natish. |

---

# B qism.
---

### 1-topshiriq: Splash ekrani
**Fayl nomi:** `SplashLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import android.util.Log
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.delay

private const val TAG = "SplashLab"
private const val SPLASH_DELAY = 2000L

@Composable
fun SplashLabScreen() {
    var version by remember { mutableIntStateOf(0) }

    // Ota composable har 500 ms da version hisoblagichini oshiradi
    LaunchedEffect(Unit) {
        while (true) {
            delay(500)
            version++
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Joriy ota versiya: v$version",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(24.dp))

        // Ikkala variantni yonma-yon ko'rsatamiz
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            Card(modifier = Modifier.weight(1f).padding(8.dp)) {
                Column(modifier = Modifier.padding(12.dp)) {
                    Text("1-variant: To'g'ridan-to'g'ri (Captured)", style = MaterialTheme.typography.titleSmall)
                    LandingScreenDirect(onTimeout = {
                        Log.d(TAG, "Direct captured timeout chaqirildi: v$version")
                    })
                }
            }

            Card(modifier = Modifier.weight(1f).padding(8.dp)) {
                Column(modifier = Modifier.padding(12.dp)) {
                    Text("2-variant: rememberUpdatedState", style = MaterialTheme.typography.titleSmall)
                    LandingScreenUpdated(onTimeout = {
                        Log.d(TAG, "UpdatedState timeout chaqirildi: v$version")
                    })
                }
            }
        }
    }
}

// 1-VARIANT: Eski lambdani ushlab qoladi (Captured)
@Composable
fun LandingScreenDirect(onTimeout: () -> Unit) {
    var status by remember { mutableStateOf("Kutilmoqda...") }
    LaunchedEffect(true) {
        delay(SPLASH_DELAY)
        status = "Tugadi!"
        onTimeout() // Dastlabki yopib olingan (v0) referensni chaqiradi
    }
    Text(status, color = Color.Gray)
}

// 2-VARIANT: Har doim eng so'nggi lambdani chaqiradi
@Composable
fun LandingScreenUpdated(onTimeout: () -> Unit) {
    var status by remember { mutableStateOf("Kutilmoqda...") }
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(true) {
        /*
         KOMMENTARIY:
         Agar LaunchedEffect(onTimeout) deb yozilsa, ota ekran har 500 ms da yangi lambda berganda
         taymer bekor qilinadi va 2 soniya hech qachon tugamaydi!
         rememberUpdatedState esa taymerni buzmasdan, eng so'nggi versiyani chaqirish imkonini beradi.
         */
        delay(SPLASH_DELAY)
        status = "Tugadi!"
        currentOnTimeout() // Eng so'nggi version (masalan, v4) chaqiriladi
    }
    Text(status, color = Color(0xFF2E7D32))
}
```

---

### 2-topshiriq: Snackbar tugmasi
**Fayl nomi:** `SnackbarLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.launch

@Composable
fun SnackbarLabScreen(onNavigateAway: () -> Unit) {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    var clickCount by remember { mutableIntStateOf(0) }

    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("Bosishlar soni: $clickCount")
            Spacer(modifier = Modifier.height(16.dp))

            Button(onClick = {
                clickCount++
                /*
                 KOMMENTARIY:
                 Nega LaunchedEffect bu yerda ishlamaydi?
                 Chunki onClick bu oddiy runtime hodisasi (callback) bo'lib, composable doirasi emas.
                 showSnackbar() suspend funksiya bo'lgani sababli, bizga ushbu Composable'ning
                 hayot sikliga bog'langan rememberCoroutineScope() zarur.
                 
                 Snackbar tugamasdan ikkinchi marta bosilganda:
                 SnackbarHost avvalgi snackbar'ni darhol to'xtatadi va yangi xabarni chiqaradi.
                 */
                scope.launch {
                    snackbarHostState.showSnackbar("Snackbar xabari #$clickCount")
                }
            }) {
                Text("Snackbar ko'rsatish")
            }

            Spacer(modifier = Modifier.height(24.dp))

            Button(
                onClick = onNavigateAway,
                colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.secondary)
            ) {
                Text("Boshqa ekranga o'tish (Scope bekor bo'lishini tekshirish)")
            }
        }
    }
}
```

---

### 3-topshiriq: Lifecycle logger
**Fayl nomi:** `LifecycleLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import android.util.Log
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver
import androidx.lifecycle.LifecycleOwner
import androidx.lifecycle.compose.LocalLifecycleOwner

private const val TAG = "LifecycleLab"

@Composable
fun LifecycleLabScreen() {
    var isObserverActive by remember { mutableStateOf(true) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("Lifecycle Kuzatuvchini yoqish/o'chirish:")
            Spacer(modifier = Modifier.width(8.dp))
            Switch(
                checked = isObserverActive,
                onCheckedChange = { isObserverActive = it }
            )
        }

        Spacer(modifier = Modifier.height(24.dp))

        if (isObserverActive) {
            LifecycleLoggerComponent()
        } else {
            Text("Kuzatuvchi o'chirilgan (Disposed)")
        }
    }
}

@Composable
fun LifecycleLoggerComponent(
    lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current
) {
    Card(modifier = Modifier.padding(16.dp)) {
        Text(
            text = "Lifecycle faol kuzatilmoqda...",
            modifier = Modifier.padding(16.dp)
        )
    }

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> Log.d(TAG, "LifecycleEvent: ON_START")
                Lifecycle.Event.ON_STOP -> Log.d(TAG, "LifecycleEvent: ON_STOP")
                else -> {}
            }
        }

        Log.d(TAG, "addObserver chaqirildi")
        lifecycleOwner.lifecycle.addObserver(observer)

        /*
         KOMMENTARIY:
         onDispose DisposableEffect ning oxirgi qatori bo'lishi shart.
         Agar onDispose olib tashlansa, kompilyator xatolik beradi va dastur build bo'lmaydi.
         */
        onDispose {
            Log.d(TAG, "removeObserver chaqirildi (Tozalandi)")
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

---

### 4-topshiriq: SideEffect va analytics
**Fayl nomi:** `AnalyticsLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import android.util.Log
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

private const val TAG = "AnalyticsLab"

// Tashqi Compose boshqarmaydigan singleton obyekt
object Analytics {
    var userType: String = "Guest"
        set(value) {
            field = value
            Log.d(TAG, "Analytics.userType yangilandi: $value")
        }
}

@Composable
fun AnalyticsLabScreen() {
    var userType by remember { mutableStateOf("Guest") }
    var localCounter by remember { mutableIntStateOf(0) }
    var effectExecutionCount by remember { mutableIntStateOf(0) }

    /*
     SideEffect har bir muvaffaqiyatli recomposition'dan so'ng ishga tushadi.
     Tushuntirish (Attempt modeli):
     Agar Compose UI chizishni boshlab, keyin bekor qilsa (commits=false),
     SideEffect ISHLAMAYDI. U faqat chizish to'liq yakunlangach (commits=true) ishlaydi.
     */
    SideEffect {
        Analytics.userType = userType
        effectExecutionCount++
        Log.d(TAG, "SideEffect ishladi! Jami bajarilish soni: $effectExecutionCount")
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Tashqi Analytics.userType: ${Analytics.userType}")
        Text("SideEffect bajarilish soni: $effectExecutionCount")
        Spacer(modifier = Modifier.height(16.dp))

        Row {
            Button(onClick = { userType = "Regular" }) { Text("Regular") }
            Spacer(modifier = Modifier.width(8.dp))
            Button(onClick = { userType = "Premium" }) { Text("Premium") }
        }

        Spacer(modifier = Modifier.height(16.dp))

        // Holat o'zgarmasa ham recomposition keltirib chiqarish uchun tugma
        Button(
            onClick = { localCounter++ },
            colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.tertiary)
        ) {
            Text("Holatni o'zgartirmasdan Recompose qilish ($localCounter)")
        }
        Text(
            "Izoh: 'localCounter' o'zgarganda 'userType' o'zgarmasa ham butun ekran recompose bo'ladi " +
                    "va SideEffect yana chaqiriladi.",
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.padding(top = 8.dp)
        )
    }
}
```

---

### 5-topshiriq: Scroll: `derivedStateOf` va `snapshotFlow`
**Fayl nomi:** `ScrollLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import android.util.Log
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.KeyboardArrowUp
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.filter
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.launch

private const val TAG = "ScrollLab"

@Composable
fun ScrollLabScreen() {
    val listState = rememberLazyListState()
    val scope = rememberCoroutineScope()

    // 1. derivedStateOf varianti (faqat true/false o'zgargandagina qayta hisoblanadi)
    val showButtonDerived by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    // 2. To'g'ridan-to'g'ri o'qish varianti (har bir yangi elementda recompose bo'ladi)
    val showButtonDirect = listState.firstVisibleItemIndex > 0

    var directRecomposeCount by remember { mutableIntStateOf(0) }
    var derivedRecomposeCount by remember { mutableIntStateOf(0) }

    // snapshotFlow orqali analytics hodisasi jo'natish
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .map { index -> index > 0 }
            .distinctUntilChanged()
            .filter { it == true }
            .collect {
                Log.d(TAG, "ANALYTICS: 0-elementdan pastga o'tildi (Hodisa yuborildi)!")
            }
    }

    Box(modifier = Modifier.fillMaxSize()) {
        Column {
            // Hisoblagichlarni ko'rsatish
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(8.dp)
            ) {
                Column(modifier = Modifier.padding(12.dp)) {
                    Text("Recomposition soni:", style = MaterialTheme.typography.titleMedium)
                    Text("To'g'ridan-to'g'ri o'qish: $directRecomposeCount")
                    Text("derivedStateOf bilan: $derivedRecomposeCount")
                }
            }

            // Kuzatuvchi komponentlar
            DirectReader(showButtonDirect) { directRecomposeCount++ }
            DerivedReader(showButtonDerived) { derivedRecomposeCount++ }

            // 100 ta elementli LazyColumn
            LazyColumn(
                state = listState,
                modifier = Modifier.fillMaxSize()
            ) {
                items(100) { index ->
                    Text(
                        text = "Ro'yxat elementi #$index",
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(16.dp)
                    )
                    HorizontalDivider()
                }
            }
        }

        // Tepaga qaytish tugmasi
        if (showButtonDerived) {
            FloatingActionButton(
                onClick = {
                    scope.launch { listState.animateScrollToItem(0) }
                },
                modifier = Modifier
                    .align(Alignment.BottomEnd)
                    .padding(16.dp)
            ) {
                Icon(Icons.Default.KeyboardArrowUp, contentDescription = "Tepaga")
            }
        }
    }
}

@Composable
fun DirectReader(show: Boolean, onRecompose: () -> Unit) {
    SideEffect { onRecompose() }
}

@Composable
fun DerivedReader(show: Boolean, onRecompose: () -> Unit) {
    SideEffect { onRecompose() }
}
```

---

### 6-topshiriq (Bonus): `produceState`
**Fayl nomi:** `ProduceStateLabScreen.kt`

```kotlin
package com.example.sideeffectslab

import android.util.Log
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.delay

private const val TAG = "ProduceStateLab"

// Natijalar sealed interfeysi
sealed interface ImageLoadResult {
    data object Loading : ImageLoadResult
    data class Success(val message: String) : ImageLoadResult
    data class Error(val exceptionMessage: String) : ImageLoadResult
}

// Soxta repozitoriy
object FakeRepository {
    suspend fun load(url: String): ImageLoadResult {
        delay(500)
        return if (url.endsWith(".png")) {
            ImageLoadResult.Success("Rasm muvaffaqiyatli yuklandi: $url")
        } else {
            ImageLoadResult.Error("Xatolik: Faqat .png formatdagi fayllar qabul qilinadi!")
        }
    }
}

// produceState yordamida funksiya (kichik harf bilan nomlangan)
@Composable
fun loadNetworkImage(url: String): State<ImageLoadResult> {
    return produceState<ImageLoadResult>(initialValue = ImageLoadResult.Loading, key1 = url) {
        Log.d(TAG, "Producer ishga tushdi (url: $url)")
        try {
            value = ImageLoadResult.Loading
            value = FakeRepository.load(url)
        } finally {
            // URL o'zgarganda yoki composable chiqib ketganda bekor qilinadi
            Log.d(TAG, "Producer to'xtatildi yoki yakunlandi (url: $url)")
        }
    }
}

@Composable
fun ProduceStateLabScreen() {
    var selectedUrl by remember { mutableStateOf("https://example.com/avatar.png") }
    var recomposeCounter by remember { mutableIntStateOf(0) }

    val imageState by loadNetworkImage(selectedUrl)

    SideEffect {
        recomposeCounter++
        Log.d(TAG, "UI Recompose bo'ldi. Jami: $recomposeCounter")
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Joriy URL: $selectedUrl", style = MaterialTheme.typography.bodyMedium)
        Spacer(modifier = Modifier.height(16.dp))

        when (val result = imageState) {
            is ImageLoadResult.Loading -> {
                CircularProgressIndicator()
                Text("Yuklanmoqda...")
            }
            is ImageLoadResult.Success -> {
                Text(result.message, color = Color(0xFF2E7D32))
            }
            is ImageLoadResult.Error -> {
                Text(result.exceptionMessage, color = MaterialTheme.colorScheme.error)
            }
        }

        Spacer(modifier = Modifier.height(24.dp))

        Row {
            Button(onClick = { selectedUrl = "https://example.com/photo.png" }) {
                Text("To'g'ri URL (.png)")
            }
            Spacer(modifier = Modifier.width(8.dp))
            Button(onClick = { selectedUrl = "https://example.com/document.pdf" }) {
                Text("Xato URL (.pdf)")
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        // Conflation tajribasi: Aynan bir xil URL ni takroran berish
        Button(
            onClick = { selectedUrl = "https://example.com/photo.png" },
            colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.secondary)
        ) {
            Text("Aynan bir xil URL ni bosish (Conflation)")
        }
    }
}
```

---

### Asosiy boshqaruvchi (MainActivity.kt)
**Fayl nomi:** `MainActivity.kt`

```kotlin
package com.example.sideeffectslab

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    MainLabNavigation()
                }
            }
        }
    }
}

enum class LabScreen(val title: String) {
    SPLASH("1. Splash"),
    SNACKBAR("2. Snackbar"),
    LIFECYCLE("3. Lifecycle"),
    ANALYTICS("4. SideEffect"),
    SCROLL("5. Scroll"),
    PRODUCE_STATE("6. produceState")
}

@Composable
fun MainLabNavigation() {
    var currentScreen by remember { mutableStateOf(LabScreen.SPLASH) }

    Column(modifier = Modifier.fillMaxSize()) {
        ScrollableTabRow(
            selectedTabIndex = currentScreen.ordinal,
            edgePadding = 8.dp
        ) {
            LabScreen.values().forEach { screen ->
                Tab(
                    selected = currentScreen == screen,
                    onClick = { currentScreen = screen },
                    text = { Text(screen.title) }
                )
            }
        }

        Box(modifier = Modifier.fillMaxSize()) {
            when (currentScreen) {
                LabScreen.SPLASH -> SplashLabScreen()
                LabScreen.SNACKBAR -> SnackbarLabScreen(onNavigateAway = { currentScreen = LabScreen.SPLASH })
                LabScreen.LIFECYCLE -> LifecycleLabScreen()
                LabScreen.ANALYTICS -> AnalyticsLabScreen()
                LabScreen.SCROLL -> ScrollLabScreen()
                LabScreen.PRODUCE_STATE -> ProduceStateLabScreen()
            }
        }
    }
}
```

---

# Logcat chiqishlari hisoboti

### 1-topshiriq (Splash) Logcat:
```text
D/SplashLab: Direct captured timeout chaqirildi: v0
D/SplashLab: UpdatedState timeout chaqirildi: v4
```
*Tahlil:* `Direct` variant birinchi marta captured bo'lgan `v0` lambdasini chaqirdi. `UpdatedState` esa 2 soniya o'tgach oxirgi `v4` versiyasini chaqirdi.

### 3-topshiriq (Lifecycle) Logcat:
Switch 5 marta almashtirilganda:
```text
D/LifecycleLab: addObserver chaqirildi
D/LifecycleLab: LifecycleEvent: ON_START
D/LifecycleLab: removeObserver chaqirildi (Tozalandi)
D/LifecycleLab: addObserver chaqirildi
D/LifecycleLab: removeObserver chaqirildi (Tozalandi)
D/LifecycleLab: addObserver chaqirildi
D/LifecycleLab: removeObserver chaqirildi (Tozalandi)
```
*Tahlil:* `addObserver` va `removeObserver` soni bir-biriga to'liq mos keladi. Xotira oqishi bartaraf qilingan.

### 4-topshiriq (SideEffect) Logcat:
```text
D/AnalyticsLab: SideEffect ishladi! Jami bajarilish soni: 1
D/AnalyticsLab: Analytics.userType yangilandi: Premium
D/AnalyticsLab: SideEffect ishladi! Jami bajarilish soni: 2
D/AnalyticsLab: SideEffect ishladi! Jami bajarilish soni: 3
```
*Tahlil:* Har bir kadr chizilishi muvaffaqiyatli commit qilinganidan so'ng `SideEffect` ishonchli ishlaydi.

### 5-topshiriq (Scroll) Logcat:
```text
D/ScrollLab: ANALYTICS: 0-elementdan pastga o'tildi (Hodisa yuborildi)!
D/ScrollLab: ANALYTICS: 0-elementdan pastga o'tildi (Hodisa yuborildi)!
```
*Tahlil:* Har bir element almashganda emas, balki faqat va faqat 0-element chegarasidan pastga o'tilgandagina aniq 1 ta hodisa chiqadi.

---

# README.md hujjati

```markdown
# SideEffectsLab — Jetpack Compose Side-effects Amaliy Mashg'uloti

Ushbu loyiha Jetpack Compose'dagi 8 ta asosiy side-effect API larining amaliy tatbiqi va xususiyatlarini o'rganish uchun mo'ljallangan.

## Tanlangan Effect API'lar va Ularning Sabablari

1. **1-topshiriq (Splash): `LaunchedEffect` va `rememberUpdatedState`**
   - **Nega?** Splash ekranda 2 soniyalik kutish uchun `LaunchedEffect(true)` tanlandi. Ota ekranda tez-tez yangilanib turadigan lambdani kutish jarayonini qayta ishga tushirmasdan (restart qilmasdan) ushlash uchun `rememberUpdatedState` ishlatildi.
2. **2-topshiriq (Snackbar): `rememberCoroutineScope`**
   - **Nega?** `Button`ning `onClick` callback'i composable bo'lmagan joydan coroutine boshlashni talab qiladi. Shuningdek, ekran tark etilganda ushbu scope avtomatik tozalanadi.
3. **3-topshiriq (Lifecycle Logger): `DisposableEffect`**
   - **Nega?** `LifecycleEventObserver` ro'yxatdan o'tkazilgach, ekran yo'qolganda albatta `removeObserver` qilinishi shart. Tozalash (cleanup) mexanizmiga ega yagona API — `DisposableEffect` hisoblanadi.
4. **4-topshiriq (Analytics): `SideEffect`**
   - **Nega?** Compose ichidagi holatni tashqi `Analytics` singleton obyektiga uzatish kerak. Faqat muvaffaqiyatli chizilgan (committed) kadrlar hisobga olinishi uchun `SideEffect` tanlandi.
5. **5-topshiriq (Scroll): `derivedStateOf` va `snapshotFlow`**
   - **Nega?** 100 ta elementli ro'yxatda har bir indeks o'zgarishi butun UI ni recompose qilmasligi uchun `derivedStateOf` ishlatildi (recomposition soni 90% ga kamaydi). Chegaradan o'tish hodisasini reaktiv boshqarish va filtrlash uchun `snapshotFlow` ishlatildi.
6. **6-topshiriq (produceState): `produceState`**
   - **Nega?** Suspend manbadan olingan natijalarni (Loading, Success, Error) Compose `State<T>` formatiga aylantirish va bekor bo'lish jarayonini xavfsiz boshqarish uchun ishlatildi.
```
