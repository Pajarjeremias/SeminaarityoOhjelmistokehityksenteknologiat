# Seminaari työ
Työn tavoitteena on Ohjelmistoprojekti 2 kurssin, Varustevahti- sovelluksen backendin kontitus ja käyttöön otto pilvipalvelussa, sekä rakentaa projektin Continuous Integration ja Continuous Delivery ratkaisuja. Käsittelen työssä Dockeria, pilviympäristöjä, CI/CD-automaatioita Github actionsien avulla ja koneoppimismallien ajamista tuotantoympäristössä. Sovellus on ryhmätyönä toteutettu mobiili laitteella toimiva kuvantunnistus ohjelma.

## Docker
Minun tehtäväkseni muodostui DevOps roolin vuoksi Backendin laittaminen palvelimelle. Meidän Backendissä on mukana AI_Model joka tunnistaa sille opetettujen mallien perusteella kuvan, jonka käyttäjä joko valitsee galleriasta tai ottaa kännykän kameralla.

Aluksi piti valita miten lähtisin toteuttamaan annettua tehtävää, joten kävin läpi eri alustat joille palvelin puolen saisi pystyyn ja päädyin Renderiin, jonkin asteisen tietämyksen ja sen tietämyksen jatko kehittämisen vuoksi. Itse backend on toteutettu SQLite:llä pythonia ja FastApia käyttäen. Tietokanta ja sen sisältämä data on kevyttä eikä sisällä montaakaan riviä koodia mutta itse kuvantunniste ohjelma on raskas, jonka vuoksi kävin eri vaihtoehtoja läpi, miten saisin mahdollisimman kevyen base imagen, jotta Docker imagesta ei tulisi liian raskas, lisätessäni Varustevahdin Backendin sinne. Tutkin eri vaihtoehtoja, kuten Pytorchia varten CUDA ajurin image mutta valitsin kuitenkin kokeiluun python:3.12-slim, sillä luin artikkelin missä kehuttiin python 3.12 (Pythonspeed. 2024) ja slim vaihtoehdon otin että se olisi riisutumpi.

### Vaihe 1 Requirements.py
Tarkistin että requirements.txt:ssä on yhdessä kaikki tarvittavat tiedot ladattavista kirjastoista, sillä AI_Modelissa ja Backendissä on omat readme.md. päädyin siihen että juuri hakemistossa olevaan requirementsiin lisäsin AI_Modelissa olevat requirementsit, jotta dockerfile löytää sen paremmin, tai oikeastaan en jaksanut alkaa sen enempään kikkailemaan sijaintien kanssa, koska se on ennenkin osoittautunut epäsuotuisaksi. 

### Vaihe 2 Dockerfile
Täytyi miettiä mitä kaikkea tarvitsen. Tärkeinpänä on python base image, jossa päädyin versioon 3.12 ja siitä slim, jotta olisi vähän riisutumpi versio, sillä tiesin että ai model tulee lisäämään imagen kokoa paljon. Alpine ei olisi käynyt sillä Koko ajan oli polttava tunne että tulisi otettava liiakseenkin pois ja löysin esimerkiksi ratkaisun, jossa linuksin päivittämisen ja asentamisen yhteydessä lisää lipun --no-install-recommends, tämä toimii niin että, jokaisessa paketissa on 3 erilaista riippuvuus pakettia: required, recommended ja suggested. Eli lipun tarkoitus on olla asentamatta recommended osiota, joka keventää Ubuntun blogin mukaan Docker imagea, jopa 60 % (Ubuntu Blog. 2019). Kirjoitin Googleen Python dockerfile löysin docker best practices (Dockerdocks. s.a.), jota hyödynsin ja otin sieltä mallia tarvittavista imagen rakennus tarvikkeista. 
Ensimäisellä image buildilla se meni läpi mutta kun yritin ajaa tuli virheilmoitus
```bash
RunTimeError: Form data requires "Python multi-part" to be installed.
```
Eli täytyy lisätä requirements.txt tiedostoon python multipart ja kokeilla uudestaa buildia. Sain imagen toimimaan menemällä osoitteeseen: [http://127.0.0.1:8080/docs](http://127.0.0.1:8080/docs)
Varustevahdin imagessa saattaa olla vielä vähän sistitävää, sillä sen kooksi muodostui 12.97 GB.

<img width="685" height="427" alt="ohkeat_lopputyo" src="https://github.com/user-attachments/assets/ce59443c-9304-4faa-9a8f-71297a4cdfb6" />

Kuten kuvasta näkee, buildissa kestää pitkään.
Ensimmäinen versio:

```dockerfile
# Base image to use, fewer included packages, because we need to keep image lighter for AI_Model and database
FROM python:3.12-slim

# All commands about to happen happens in this directory
WORKDIR /workspace

# Linux commads, for upgrade and install, flags first mean yes and second not to install "recommended dependencies packages" making it lighter. 
# Source: "https://ubuntu.com/blog/we-reduced-our-docker-images-by-60-with-no-install-recommends"
RUN apt-get update && apt-get install -y --no-install-recommends && rm -rf /var/lib/apt/lists/*
    # Deletes caches, for again making image lighter (source: https://docs.docker.com/build/building/best-practices/#run)
(ei toimi tälläisenään, kommentit pitää siirtää pois tieltä)

# Copying file to image, so pip can install
COPY requirements.txt .

# install and upgrade pip and then install requirements file
RUN python -m pip install --upgrade pip && pip install -r requirements.txt

# copy app to image
COPY app ./app

# copy AI_MODEL to image
COPY AI_Model ./AI_Model

# making folder for save_upload to save pictures, -p flag creates them if they do not already exist.
RUN mkdir -p /app/uploads

# Database path
ENV DATABASE_URL=sqlite:///./varustevahti.db

# Using uvicorn to start application, listening everywhere and use Render given port or if not given then port 8000
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]
```
### vaihe 3 Palvelin
Olin julkaisemassa Varustevahdin backendiä Renderille, kunnes tajusin että backend kuvantunnistus ohjelmineen ei tule pyörimään ilmais version 512 mb RAMilla ja 0.1 CPUlla. Muistin ohjelmistoprojekti 1 - kurssilla Markku Ruonavaaran maininneen CSCn tarjoamien palveluiden, kuten Rahti tarjoavan opiskelijoille paljon teknisiä resursseja, joten päätin kokeilla tätä. Onnistuin luomaan projektin, mutta loin sen aluksi käyttäen Pythonin builder imagea, kunnes tajusin ettei se missään kohtaa lue Docker imageani. Tämän tajuttuani loin uuden projektin käyttäen tehtyä Dockerfileä, kaatui heti ja huomasin käyttäneeni pientä d kirjainta tiedostossa, korjasin asian ja build onnistui mutta podin käynnistyksessä ilmeni ongelma 
```bash
AvailableFalseMinimumReplicasUnavailableDeployment does not have minimum availability
```
Podin logi kertoi ettei se pysty avaamaan ja antoi samalla virheilmoitukseen viittaavan sivuston virheelle. Sieltä ei löytynyt mitään minua auttavaa, joten etsin haku koneista virhe ilmoituksella mutta mitään ei löytynyt, jolloin menin CSCn FAQ Rahti osioon, josta kävin kaikki kysytyt kysymyset läpi ja viimeisenä oli kysymys "Why this container report permission denied errors?" jossa todettiin:

"The most common reason for a container to fail to run in Rahti, is that the container image expects to be run as root. As Rahti does not run images as root, permission denied errors will stop the execution" ja toinen teksti "the other option is to modify the current image to make it work with a non root user, for example as described in the Creating images section."(Docs CSC. 2025).

Creating image section ei auttanut minua sillä siellä käytettiin Alpinea joka käyttää työkaluna aptn sijasta kevyempää apk paketin hallinta työkalua, eikä se ole minulle tuttu.
Joka tapauksessa Rahti ei aja kontteja root käyttäjänä ja keskeyttää ajon. Rahdin käyttämä Kubernetesin OpenShift palvelu estää turvallisuus syistä root käyttäjän käytön, luomalla satunnaisia käyttäjätunnisteita(Random UID) tietyn numero skaalan välillä. Siksi minun tuli muuttaa konttien hakemisto oikeuksia, siten että random UID pystyy kirjoittamaan SQLite tiedostoihin. Löysin RedHatBlogin, jossa kerrotaan että kontit kuuluvat oletuksena ryhmään 0, joka on root group(Red Hat Blog. 2020, kohdan UIDs and GUIs in Action (Container perspective) lopussa), mikä tarkoittaa sitä että vaikka kontit ei saa käyttää root käyttäjää, se voi silti kuulua root ryhmään ja hyödyntää siten ryhmä oikeuksia.
Jolloin tajusin että minun tulee Dockerfilessä luoda uusi hakemisto /workspace, jolle annetaan group 0, jotta random UID:lla ajettava kontti pääsee käsiksi SQLiteen. Joten loin workspace hakemiston ja muutin sen groupiksi 0 eli root group (chgrp -R 0 / workspace), jonka jälkeen annoin tälle ryhmälle samat oikeudet kuin tiedoston omistajalla (chmod -R g=u /app). Pohdiskelu apua ja henkistä tukea aiheesta sain pikku serkultani Developer Adam Wallinilta mutta tiedon haun ja lopullisen selvityksen tein itse. Tämän jälkeen Backend rupesi pyörimään osoitteessa: [https://backend-git-varustevahti.2.rahtiapp.fi](https://backend-git-varustevahti.2.rahtiapp.fi).

Ongelmat eivät jääneet tähän vaan tämän jälkeen ryhmäläinen lähetti viestin:

**ainakaan kuvan lataus ei toiminut tuo auto toiminnon kanssa. tuli vaan virhe:**
```json
{
"detail": "Classification failed: Failed to download weights for tag 'openai': Failed to download file (open_clip_pytorch_model.bin) for timm/vit_large_patch14_clip_224.openai. Last error: [Errno 13] Permission denied: '/.cache'"
}
```
DevOpsin arkipäivää ja sankari tulee apuun. selvitin mitä polkuja AI_Model yrittää kirjoittaa välimuistinsa ja lisäsin ne aikaisemmin SQLiten oikeus ongelmia varten lisättyyn Dockerfilen OpenShiftin rootin kiertämis komentoon. Tämä ei vielä riittänyt sillä malli yritti kirjoittaa latausvaiheessa välimuistin /.cache hakemistoon, johon ei ole kirjoitusoikeuksia Rahdissa. Tästä syystä kovakoodasin linuxin yleisen välimuisti polun ja Torchin välimuistin menemään hakemistoon /workspace/.cache.(PyTorch. 2025. kohta Where are my downloaded models saved).
torch viittaa pytorchiin, joka on viitekehys ja ekosysteemi pythonin syväoppimiselle.

Ongelmat ei jääneet siihenkään vaan tuli virheilmmoitus 502 bad gateway, jolloin huomasin että Rahdin pod kaatuu samantien kun tekee haun items/auto (kuvan tunnistus) endpointiin. Kävin läpi podin tietoja ja löysin podin container detailsin sisältä last state OOMKilled, joka meinaa Linuxin pysäyttäneen kontin, koska se on ylittänyt sille varatun muistirajan (Komodor.2021). Menin tarkistamaan Rahdin minulle myöntämät resurssit, jotka löysin Developer -> project -> AppliedClusterResourceQuotas -> YAML ja ne näytti riittävän kuvantunnistuksen pyörittämiseen.

<img width="464" height="235" alt="rahtiresurssittotal" src="https://github.com/user-attachments/assets/1e58d6dc-36b1-47ef-bd79-001354a1ae3b" />
<img width="460" height="175" alt="rahtiresurssittotal2" src="https://github.com/user-attachments/assets/afc16ae4-1d04-4ef0-a88a-9a2c7c907589" />

Joten nyt minun pitää etsiä missä pystyn muuttamaan podin käynnistämiseen käytettäviä resursseja, jotka löysin Administrator -> Deployments -> Deployment details -YAML. Siellä on kohta resources: {}, jonka päälle mentäessä se sanoo että ei saa muuttaa mutta täältä löytyy lisätietoja (Kubernetes. 2025) ja sain annettua podille lisä resursseja.

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
    ephemeral-storage: "512Mi"
  limits:
    cpu: "1"
    memory: "4Gi"
```
ja muutaman kirjoitus virhe korjauksen jälkeen sain sen tallennettua, se käynnisti podin rolloutin automaattisesti, jolloin uudelleen kokeillessani FastApi interactive docsissa items/auto, se toimi.

tässä matkan varrella muunneltu Dockerfile joka toimii:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /workspace

ENV HOME=/workspace
# Linux user´s directory path for non-essential, cached data
ENV XDG_CACHE_HOME=/workspace/.cache
# Pytorch cache env path
ENV TORCH_HOME=/workspace/.cache/torch

COPY requirements.txt .

RUN python -m pip install --upgrade pip && pip install --no-cache-dir -r requirements.txt

COPY app ./app

COPY AI_Model ./AI_Model

# making folder for save_upload to save pictures, -p flag creates them if they do not already exist.
RUN mkdir -p /workspace/uploads /workspace/.cache/torch \
 && chgrp -R 0 /workspace /.cache \
 && chmod -R g=u /workspace /.cache
#Changed workspaces group to same as OpenShifts default root group (0) and the gave group file owner permissions for write access
# Database path
ENV DATABASE_URL=sqlite:///./varustevahti.db

EXPOSE 8080
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

En muista tasan tarkkaan missä kohtaa tajusin ettei
```bash
RUN apt-get update && apt-get install -y --no-install-recommends && rm -rf /var/lib/apt/lists/*
```
komentoa tarvita ollenkaan, sillä ei ole mitään mitä Linuxilla pitäisi tähän asti käsitellä. Ja näin sain pienennettyä imagea.

### Hienosäätö

Datasta piti saada pysyvää, joten etsin googlesta how to make data persistent in rahti ja löysin tämän dokumentaation (Docs CSC. 2025). Tätä kun hetken aikaa pyöritti, niin sain datan pysyväksi rahtiin. Tämä oli nimittäin hyvä hoitaa kuntoon ennen kuin julkaisen käyttöliittymän.
Nyt kun olin kaiken saanut toimimaan, halusin tehdä pieniä viilauksia. Docker Imagen 12 GB koko pyöri mielessä että se täytyy saada pienemmäksi. Muistin että DevOps kurssilla käytettiin ainakin Docker Compose tiedostoa mutta en nähnyt järkevää tapaa hyödyntää sitä muuten kuin jakamalla SQLiten ja AI_Modelin mutta epäilin vaikutuksen olevan pieni, sillä meidän tietokanta on muutenkin kevyt. Minulle mainittiin Docker Multi-stage buildista, Dockerin dokumentaatiota lukiessa mietin tämän sopivan tähän projektiin. Yritin saada sitä toimimaan mutta en onnistunut, sillä tuli muita asioita projektissa tehtäväksi, jonka vuoksi tämä jäi hiomatta kuntoon. Mielenkiinnosta aion omalla ajalla löytää tavan, jolla imagen kokoa saa pienennettyä.

Rahdissa tuli vielä ongelma, sain ryhmäläisiltä ilmoituksen, jossa eräs kertoi työntäneensä commitin mainiin, jossa oli lisätty muutama endpoint, jonka jälkeen tietyt toiminnot ei toiminut ja virheilmoituksessa oli 500 internal server error. Katsoin rahdin logeja ja siellä oli teksti:
```bash
usr/local/lib/python3.12/site-packages/pydantic/_internal/_config.py:383: UserWarning: Valid config keys have changed in V2:
* 'orm_mode' has been renamed to 'from_attributes'
```
Vaihdoin nuo config keyesit oikeiksi ja työnsin commitin mainiin, jotta se uudelleen rakentuisi. Se rakentui ongelmitta mutta podissa sama ongelma jatkoi ja kun kävin katsomassa FastAPIn interactive docsista, niin myöskään uudet endpointit eivät tallentuneet. Tarkistin Rahdin tietoja ja siellä luki että rahdissa on oikea commit, joten päädyin lopputulokseen että ongelma on Rahdin päässä. Tutkin ja tutkin kaikki openshiftin tekemät image hashien yhteydet ja kolusin läpi jokaisen kohdan josta voisi löytyä jotain viitettä ongelmaan. Kolmen tunnin selvittelyn jälkeen menin topologyyn buildin kohdalla ja tein podiin restart rolloutin, tämän jälkeen se toimi normaalisti. Olin ollut koko ajan siinä oletuksessa että podi käynnistyy uudella imagella aina kun tekee uuden buildin mutta ilmeisesti näin ei ole.

Lisäksi tein myös huomion, olin tehnyt workflowin jatkuvalle julkaisemiselle Rahtiin aina kun repositorion mainiin lisätään commit. Tein projektin dokumentaatiota readme tiedostoon, suoraan githubissa ja siinä tuli todella monta committia, kun tajusin mitä tapahtuu ja kävin katsomassa Rahtia siellä oli noussut buildien määrä 17:sta 54:ään.

## CI/CD Pipeline
Tulee mieleen aika vähän asioita mitä voin tehdä, koska devaaminen tapahtuu Expo Golla ja julkaisua ei voi tehdä koska Android Play kauppaan ja Applen App Storeen julkaiseminen maksaa ja Expo Hosting ei onnistunut meillä ilman ongelmia, sillä React ja React-Dom olivat eri versioina, joka loi ristiriitoja React Native Webin asentamisessa.

### Github actions
Tein projektiin auto deploy webhookin, joka tekee rahtiin uuden buildin aina kun pushataan Githubissa mainiin. Ajattelin että tämä on toki automatisointia mutta haluan devops näkulmasta pystyä tarpeen tullen muuttamaan sitä miten haluan, joten päätin että teen sen mielummin github actionina. Muistin Devops kurssilta ja työharjoittelusta että .yml tiedostoon voi lisätä Bash skriptausta. Se ei ollut ihan niin suora viivaista kun ajattelin, eli sanomalla vain että tähän osoitteeseen, vaan virhe ilmoituksesta selvisi että GET-pyynnön sijaan täytyykin tehdä POST ja siihen tulee vielä sisällyttääkkin tietoa nimittäin content type (Docs CSC. 2025). Ei se siitä johtunut vaan oli käytössä väärä rahdin secret, nimittäin /github päätteinen vaikka piti olla /generic. (https://github.com/openshift/origin/issues/4458).

#### Continuous Delivery (CD)

```yaml
name: Rahti deployment on push to main
on:
  push:
    branches: 
      - main

jobs:
  deployment:
    runs-on: ubuntu-latest
    steps:
      - name: deployment
        env:
          deploy_url: ${{ secrets.RAHTI_WEBHOOK_URL }}
        run: |
          curl -X POST  "$deploy_url"
```
Se oli CDn osalta siinä, haluan vielä tehdä CI osiollekkin jotain. Loin projektin frontendiin Expo lint ja doctor toiminnot aina kun pushaa mainiin ja hyvä niin sillä lint kohdalla tuli 27 error ja 48 varoitusta. 

#### Continuous Integration (CI)

```yaml
name: dependencies install correctly and diagnosing expo project

on:
  push:
    branch:
      - main

jobs:
  ci-doctoring:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v5
      - name: Setting up Node
        uses: actions/setup-node@v6
        with:
          node-version: 24
      - name: Installing dependencies
        run: npm ci
      - name: linting
        run: npx expo lint
      - name: Diagnosing expo
        run: npx expo doctor
```
Tuntui että jotain pitäisi vielä saada tuotettua ja halusin kokeilla että onnistunko github actioneita käyttämällä tarkistamaan Rahdin buildin onnistumisen. Suurin ongelma ehkä on se että jokainen build kestää noin 16 minuuttia, joten yml tiedostoon tulee käyttää bash scriptiä sivuston pingaamiseen ja kun build onnistuu, niin testaamaan että sen tärkein kuvantunnistus toiminto toimii.

### Eas workflows
Eas Workflowit on React Native CI/CD palvelu, jotka on tehty helpoittamaan sovelluksen julkaisemista. Ne tarjoaa valmiiksi määriteltyjä työkulkuja, joilla voi muun muuassa rakentaa, julkaista, päivittää sovellusta ja ajaa maestro testejä. Kaikki työtyypit ajetaan EAS palvelussa, joten täytyy vain hallita yhtä YAML tiedostojen kokonaisuutta. Kaikista ajokerroista syntyvät artifaktit näkyvät yhdessä paikassa expo.devissä. (Expo. 2025)

Muut CI palvelut, kuten Github Actions, ovat yleiskäyttöisempiä ja mahdollistavat enemmän. Niissä joudut kuitenkin ymmärtämään syvemmin jokaisen jobin toteutuksen. EAS Workflows puolestaan tarjoaa yleisimmät mobiilisovellusten julkaisu ja build työt valmiiksi paketoituina, jolloine ne saa nopeasti käyttöön. Työnkulut käyttävät myös tehtävään optimoituja pilvikoneita, joita expo päivittää jatkuvasti. (Expo. 2025)

Omasta mielestäni EAS workflowien käyttö on erittäin hyödyllistä mobiili sovellusten rakentajille ja eas cli tarjoaa paljon hyödyllisiä ominaisuuksia kehityksessä.

Koska emme julkaise expo sovellusta minnekkään ja kaikki dokumentaatio tästä aiheesta on pelkästään buildausta varten, tein alustavan workflowin, joka voisi toimia jos joskus joku haluaa Varustevahdin expo osuuden julkaistavaksi. 

Kun ottaa EAS workflowit käyttöön tulee tehdä muutama asia ennen kuin kirjoitetut yml tiedostot saa toimintaan:
1. Täytyy kirjautua Expo accountille
2. Luoda projekti
   ```bash
npx create-expo-app@latest
   ```

3. Linkittää EAS projekti, lokaaliin projektiin
   ```bash
npx eas-cli@latest init
   ```

4. Lisätä eas.json projektin juureen.
   ```bash
touch eas.json && echo "{}" > eas.json
   ```

Tämän luodaan kansiot .eas ja workflow, .eas/workflows, jonne voidaan alkaa luomaan workflow tiedostoja yml muodossa
Tämä workflow rakentaa rinnakkain Android ja iOS julkaisu versiot projektista. Tähän voisi myös lisätä automaattisen julkaisun sovellus kauppoihin.

```yml
# Source: https://www.youtube.com/watch?v=OJ2u9tQCpr4
name: Create Production Build

jobs:
  build_android:
    type: build
    params:
      platform: android
  build_ios:
    type: build
    params:
      platform: ios

# To run this workflow, in terminal use this command:
# npx eas-cli@latest workflow:run production-builds.yml 

#Use this if you want it to be automated:
#on:
#  push:
#    branches: ['main']

#  jobs:
#    build_android:
#      type: build
#      params:
#        platform: android
#    build_ios:
#      type: build
```
Tämä taas rakentaa Androidille kehittämis buildin ja iOS:lle kehittämist buildin ja simulaattorin.

```bash
# source: https://www.youtube.com/watch?v=OJ2u9tQCpr4
name: development builds

jobs:
  android_dev_build:
    name: android build
    type: build
    params:
      platfrom: android
      profile: development
  ios_device_dev_build:
    name: iOS device build
    type: build
    params:
      platform: ios
      profile: development
  ios_simulator_dev_build:
    name: iOS simulator build
    type: build
    params:
      platform: ios
      profile: development-simulator
# To run this workflow, in terminal use this command:
# npx eas-cli@latest workflow:run developments-builds.yml 
```
(Expo. 2025)

## Oppimani asiat
Työn aikana opin käytännössä, mitä DevOps rooli tarkoittaa oikeassa projektissa. Siinä täytyy ottaa huomioon kaikki eri järjestelmien, palveluiden, ohjelmointi kielten ja teknologioiden vaikutukset toisiinsa. Mukavana lisänä oli se että projektissa mihin nämä kaikki implementoin, oli mukana ML malli. Opin itse asiassa seminaarii työssä todella paljon.

Docker osiossa, opin luomaan Python pohjaisen Docker imagen, jossa backend ja ML malli on samassa paketissa. Perehdyi myös eri image vaihtoehtoihin ja siihen miten --no-install-recommends, välimuistien poistaminen ja requirementsien siistiminen vaikuttaa imagen kokoon. Tämän lisäksi pääsin käytännössä huomamaan ettei Dockerfilen rakentaminen ole vältämättä niin suora viivaista, sillä loppujen lopuksi jouduin otamaan huomioon asoita kuten välimuistit, hakemistojen polut ja käyttöoikeudet.

Suurin oppi liittyi rahtiin ja sen käyttämään OpenShiftiin, sillä pääsin syventymään sen järjestelmän tapoihin toimia taustalla, kuten ettei kontteja ajeta root käyttäjänä, vaan se generoi random UIDn, joka taas esti SQLiten ja ML mallin välimuistien kirjoittamisen. Opin todella paljon ongelmien ratkomisesta, mistä tietoa etsiä ja miten sitä tietoa hyödyntää käytännössä. Lisäksi rahdissa pääsin käsittelemään resurssien hallintaa, miten podien ja buildien välinen yhteys toimii sekä mistä voin löytää palvelun tarjotamista logeista tärkeää tietoa, joilla saan ongelman ratkottua tai edes yhteyden mitä voin miettiä pidemmälle ratkaisun löytämiseksi. 

CI/CDstä opin miten githubiin tiettyyn haaraan laittamalla commitin, saa Rahdissa buildin käynnistettyä, käyttämättä Rahdin tarjoamaa webhook ratkaisua. Koen että muutenkin sain Varustevahdin tähän vaiheeseen kaiken tarpeellisen tällä osa-alueella.

## Jatkokehitys ideat
Lähinnä jatkokehittäisi Docker imagea pienemmäksi multi-stagella ja kuvantunnistusta rahtiin julkaistulla backendillä nopeammaksi. Lisäksi olisi hienoa saada sovelluksen expo osuus sovellus kauppoihin ja eas workflowien avulla saada sen automaatisoitua sekä saada github actionsillä CI putkeen testejä.

Ideana voisi myös olla pohtia että olisiko järkevä erottaa ML malli erilliseksi mikropalveluksi, jolloin ainakin pysyisi paremmin kärryillä ongelma tilanteessa että kummassa on ongelma backendissä vai ML mallissa.

Rahdin osalta olisi myös hyvä selvittää pystyykö siellä tehdä asioita nopeuttamaan sovellusta, esimerkiksi lisäämällä ML mallille lisää GPUta 

## Lähteet

Docs CSC. 29.01.2025. Why this container report permission denied errors?. Luettavissa: https://docs.csc.fi/support/faq/why-this-container-does-not-work. Luettu: 30.11.2025.

Docs CSC. 08.04.2025. Persistent volumes. Luettavissa: https://docs.csc.fi/cloud/rahti/storage/persistent/. Luettu: 7.11.2025

Docs CSC. 17.02.2025. Webhooks. Luettavissa: https://docs.csc.fi/cloud/rahti/tutorials/webhooks/. Luettu: 12.11.2025

DockerDocks. s.a. best practices RUN. Luettavissa: https://docs.docker.com/build/building/best-practices/#run. Luettu: 29.10.2025

Expo Docs. 22.07.2025. Get started with EAS Workflows. Luettavissa: https://docs.expo.dev/eas/workflows/get-started/. Luettu: 17.11.2025

Komodor. 23.09.2021. How to Fix OOMKilled Kubernetes Error (Exit Code 137). Luettavissa: https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/. Luettu 2.11.2025.

Kubernetes. 06.08.2025. Recourse Management for Pods and Containers. Luettavissa: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/. Luettu: 2.11.2025

Pytorch. 13.06.2025. torch.hub. Luettavissa: https://docs.pytorch.org/docs/stable/hub.html. Luettu: 1.11.2025

pythonspeed. 15.10.2024. The best Docker base image for your Python application. Luettavissa: https://pythonspeed.com/articles/base-image-python-docker-images/. Luettu: 27.10.2025

Red Hat Blog. 28.07.2020. A Guide to OpenShift and UIDs. Luettavissa: https://www.redhat.com/en/blog/a-guide-to-openshift-and-uids. Luettu: 31.10.2025.

Ubuntu Blog. 15.11.2019. We reduced our Docker images by 60% with -no-install-recommends. Luettavissa: https://ubuntu.com/blog/we-reduced-our-docker-images-by-60-with-no-install-recommends. Luettu: 27.10.2025 
