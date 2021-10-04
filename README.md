    Notlar videolardan bakilarak dogrudan yazilmis, tekrardan calistirilmamistir. Bu sebeple bazi cikti ornekleri (token gibi) gelisi guzel yazilmis olabilir veya yazim yanlisi sebebiyle hata olusabilir. Hatalari duzeltmek, metindeki Turkce karakterleri veya baska bir duzenleme yapmak isterseniz pull request olusturabilirsiniz :)


- [3.1 Proje Kurulumu ve UserProfiles API](#31-proje-kurulumu-ve-userprofiles-api)
  - [3.1.1 Proje Kurulumu](#311-proje-kurulumu)
  - [3.1.2 Modellerimiz](#312-modellerimiz)
- [3.3 Sinyaller - Django Signals](#33-sinyaller---django-signals)
- [3.4 Serializerlarimiz](#34-serializerlarimiz)
  - [Aciklamalar:](#aciklamalar)
  - [Basic Authentication](#basic-authentication)
  - [Token Authentication](#token-authentication)
  - [DRF - Session Authentication](#drf---session-authentication)
    - [Nasil Calisir?](#nasil-calisir)
  - [JSON Web TOkens (JWT)](#json-web-tokens-jwt)
- [Django Rest Auth - Bolum 1](#django-rest-auth---bolum-1)
    - [Clientlarimiz (Istemcilerimiz)](#clientlarimiz-istemcilerimiz)
    - [Ornek Bir View Uzerinden Authentication Sistemimizi Test Edelim](#ornek-bir-view-uzerinden-authentication-sistemimizi-test-edelim)
- [Django Rest Auth - Bolum 2 - Registration](#django-rest-auth---bolum-2---registration)
  - [3.7.1 Ayarlamalar](#371-ayarlamalar)
  - [3.7.2 URL'lerimiz](#372-urllerimiz)
- [3.8 ViewSets ve Routers - Part 1](#38-viewsets-ve-routers---part-1)
- [3.9 ViewSets ve Routers - Part 3](#39-viewsets-ve-routers---part-3)
- [3.10 ViewSets ve Routers - Part 3](#310-viewsets-ve-routers---part-3)
- [3.11 ViewSets ve Routers - Part 4](#311-viewsets-ve-routers---part-4)
- [3.12 DRF'de Filtreleme](#312-drfde-filtreleme)

Bu seride, Django RestFramework'te daha ileri konulari ogrenecegiz. Bu tutorial sonunda asagidaki konulari ogrenmis olacagiz:
- DRF'teki Temel Authentication Metodlari
- Rest servisleri ile Kayit ve Authentication
- Django-REST-Auth konusunda temel bilgiler
- Viewset ve Router siniflari
- DRF'te filtreleme islemleri
- User modelimizi genisleterek bir Profile modeli olusturma (one-to-one relationship)

# 3.1 Proje Kurulumu ve UserProfiles API
Bu dersimizde Kullanici profili temali bir proje olusturacagiz ve bu sezondaki tum derslerimizi bu projemiz uzerinden isleyecegiz. Su ana kadar, `django.contrib.auth.models` ile gelen `User` modelini kullanmistik. Bu tutoria serisinde ise bu Kullanici modelini yeni bir profil modeli olusturarak genisletecegiz. Boylelikle Kullanici modelimize yeni alanlar ekleme sansimiz olacak (firma, biyografi, profil resmi vb.). Ayrica sosyal medya uygulamalarinda oldugu gibi kullanicilarin profillerine durum mesaji verebilmesini saglayacagiz.

Durummesajlarini yazarken de `django.signals` (sinyaller) sistemine de giris yapmis olacagiz. Sinyaller, Django icerisinde yer alan en guclu sistemlerden birisi, cunku bazi verici fonksiyonlar ve alici fonksiyonlar ile Framework'umuzun herhangi bir yerinde bazi islemleri otomatik olarak gerceklestirebiliriz. Yani, bir islem yapildiginda otomatik olarak baska bir islemin de gerceklestirilmesi gibi. UserProfileAPI'imizi cagirdigimizda yani bir User olusturdugumuzda otomatik olarak o User'a bagli bir profil olusturulmasi islemini yapacagiz. Bu islem farkli yollarla da gerceklestirilebilecek olsa da signals konusuna girmenin ilerisi icin buyuk faydalar olusturacagina inaniyorum.

## 3.1.1 Proje Kurulumu
Yeni bir klasor icerisinde sanal Python ortami olusturup aktiflestirelim. Yuklenecek paketler icin komut:
`pip install django django_extensions djangorestframework pillow ipython`

Django proje init:
`django-admin startproject core`

Proje dizinine gecis:
`cd core`

Profiller uygulamasinin olusturulmasi
`python manage.py startapp profiller`

VS Code baslatma:
`code .`

*core/settings.py*

```python
...

INSTALLED_APPS = [
    ...
    "profiller.apps.ProfillerConfig",
    "rest_framework",
    "django_extensions",
]

...

MEDIA_URL = "/media/"
MEDIA_ROOT = "uploads"
```

Projede kullanilacak resim dosyalari icin url ve media root ayarlamalari da yapilmasi gerekmekte. Gelistirme asamasinda calisabilmesi icinse:

*core/urls.py*
```python
from django.conf import settings
from django.conf.urls.static import static
from django.contrib import admin
from django.urls import path
from django.urls.conf import include

urlpatterns = [
    path("admin/", admin.site.urls),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## 3.1.2 Modellerimiz

Bu projede iki model ile calisacagiz. 1. modelimiz Profil modeli, User modelinden turetecegiz. Durum mesajlari icin de ayri bir model olusturacagiz. User medilini genisleterek olusturacagimiz Profil modeli de User yani kullanici kayitlarimiza firma, sehir, profil resmi gibi yeni detaylar ekleyebilecegiz. Daha sonra da olusturdugumuz profil modelini durum modeline baglayacagiz.

Modellerimiz asagidaki gibi olacak. Bildigimiz uzere biz modellerimizi admine kaydettigimizde Django otomatik olarak sonuna bir **s** getirerek cogul halini ayarliyor. Turkce olmasi icin Meta sinifinda degisiklik yapacagiz.

*profiller/models.py*
```python
from django.db import models
from django.contrib.auth.models import User

class Profil(models.Model):
    user = models.OnetoOneField(User, on_delete=models.CASCADE, related_name="profil")
    bio = models.CharField(max_lenght=300, blank=True)
    sehir = models.CharField(max_lenght=120, blank=True)
    foto = models.ImageField(null=True, blank=True, upload_to="profil_fotolari/%Y/%m/")

    class Meta:
        verbose_name_plural = "Profiller"

    def __str__(self):
        # AbstractUser modeli ile olan baglantidan dolayi
        # username erisimi asagidaki sekilde
        return self.user.username


class ProfilDurum(models.Model):
    user_profil = models.ForeignKey(Profil, on_delete=models.CASCADE)
    durum_mesaji = models.CharField(max_length=240)
    yaratilma_zamani = models.DateTimeField(auto_now_add=True)
    guncellenme_zamani = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name_plural = "DurumMesajlari"
    
    def __str__(self):
        return str(self.user_profil)
```

Yuklenen profil resimlerini tekrar olceklendirerek sunucudaki yuku azaltalim. Profil modelinin save methodunda bazi degisiklikler yapacagiz.


*profiller/models.py*
```python
from django.db import models
from django.contrib.auth.models import User

from PIL import Image

class Profil(models.Model):
    user = models.OnetoOneField(User, on_delete=models.CASCADE, related_name="profil")
    bio = models.CharField(max_lenght=300, blank=True)
    sehir = models.CharField(max_lenght=120, blank=True)
    foto = models.ImageField(null=True, blank=True, upload_to="profil_fotolari/%Y/%m/")

    class Meta:
        verbose_name_plural = "Profiller"

    def __str__(self):
        # AbstractUser modeli ile olan baglantidan dolayi
        # username erisimi asagidaki sekilde
        return self.user.username

    def save(self, *agrs, **kwargs):
        super().save(*agrs, **kwargs)
        if self.foto:
            img = Image.open(self.foto.path)
            if img.height > 600 or img.width > 600:
                output_size = (600, 600)
                img.thumbnail(output_size)
                img.save(self.foto.path)
```

Migrationlarin yapilmasi:

`python manage.py makemigrations`
`python manage.py migrate`

Superuser olusturulmasi:
`python manage.py createsuperuser`

*profiller/admin.py*
```python
from django.contrib import admin
from profiller.models import Profiller, ProfilDurum

admin.site.register(Profil)
admin.site.register(ProfilDurum)
```

3.2'yi nerde gectik :D
# 3.3 Sinyaller - Django Signals
    ON NOT: Django sinyaller ile bir user olusturuldugunda otomatik olarak user'a bagli bir profil olusturulmasi. Bu islemi viewlerimiz icerisinde de yapabiliriz `profil.save(commit=False)` devaminda yeni user nesnesinin profile tanimlanmasi ve `profil.save(commit=True)` ile profil kaydetme isleminin tamamlanmasi gibi bir akis izlenebilir.

    Bu islemi is mantigi akisi geregi her user olusturulmasinda tekrar edecegimiz icin otomatik hale getirmemiz cok daha mantikli ve DRY (Don't Repeat Yourself) bir yaklasim.

Kullanici profil olusturma islemini `django_shell`'de yapacak olursak takip edilecek adimlar:

```terminal
> python manage.py shell_plus
> from django.contrib.auth.models import User
> from profiller.models import Profil
> user = User(username='testuser', password='testing321.')
> user.save()
## Bu asamada yeni user nesnesi (instance) olusturuldugunda yeni profil olusturulmasi icin bir sinyal gondermemiz lazim.
## Gonderecegimiz sinyalle de asagidaki islemleri yapmamiz lazim
> Profil.objects.create(user=user)
```

Bu islemin her defasinda tetiklenebilmesi icin bir sinyal olusturalim.
*profiller/signals.py*
```python
from django.contrib.auth.models import User
from profiller.models import Profil
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_profil(sender, instance, created, **kwargs):
    print(instace.username, "__Created: ", created) # Yeni profil olustugunda bool cikti alacagiz
    if created:
        Profil.objects.create(user=instance)
```

Devaminda da *profiller/apps.py* dosyasi icinde olusturdugumuz bu sinyal fonksiyonunu import etmeliyiz ama DOGRU YERDE:
*profiller/apps.py*
```python
from django.apps import AppConfig

class ProfillerConfig(AppConfig):
    name = "profiller"

    def ready(self):
        import profiller.signals
```

Normalde, ileride hata alinmasininonune gecmek icin *profiller/__init__.py* dosyasi icerisine `default_app_config = "profiller.apps.ProfillerConfig"` satirini eklememiz gerekirdi. *settings.py* dosyasinda `INSTALLED_APPS` altinda uygulamamizi kaydettigimiz icin gerek kalmadi.


```terminal
> python manage.py shell_plus
> from django.contrib.auth.models import User
> from profiller.models import Profil
> user = User(username='testuser_2', password='testing321.')
> user.save()
testuser_2 __Created: True
```

Sinyaller konusunu pekistirmek icin soyle bir hikaye daha yazalim. Bir kullanici sisteme kayit oldugunda ilk profil durum mesajini da otomatik olarak yayimlasin. 

*profiller/signals.py*
```python
@receiver(post_save, sender=Profil)
def create_ilk_durum(sender, instance, created, **kwargs):
    if created:
        ProfilDurum.objects.create(
            user_profil=instance,
            durum_mesaji=f"{instance.user.username} klube katildi",
        )
```
Ikinci fonksiyonumuzda `sender` profil oldu, cunku biz instance argumani ile otomatik olarak `ProfilDurum` modeli icerisine ilgili profili vermek durumundayiz.

# 3.4 Serializerlarimiz

Ilk olarak profiller klasoru icerisinde api isimli bir klasor olusturacagiz. Permissions, api'ler ile ilgili view'lerimiz ve bu gibi scriptlerimizin tamamini bu klasor icinde toplayacagiz. *profiller/api* klasorunu olusturduktan sonra bu klasor icerisinde *serializers.py* dosyamizi olusturup ModelSerializer sinifini kullanarak hizli bir sekilde serializer'larimizi olusturacagiz.

```python
from profiller.models import Profil, ProfilDurum
from rest_framework import serializers

class ProfilSerializer(serializers.ModelSerializer):
    user = serializers.StringRelatedField(read_only=True)
    foto = serializers.ImageField(read_only=True)

    class Meta:
        model = Profil
        fields = "__all__"

class ProfilFotoSerializer(serializers.ModelSerializer):

    class Meta:
        model = Profil
        fields = ("foto",)

class ProfilDurumSerializer(serializers.ModelSerializer):
    user_profil = serializers.StringRelatedField(read_only=True)

    class Meta:
        model = ProfilDurum
        fields = "__all__"
```

## Aciklamalar:

- Oncelikle Authenctication ve Permissions yani izinler sistemini karistirmamamiz lazim.
- Permissions yani izinler konusunda hatirlarsaniz, baglanan kullanicinin login/logout olup olmadigina, login olduysa admin, staff ya da standart bir kullanici olup olmadigina bakmistik.
- Bu yuzden authentication yani yetkilendirme islemini sadece login/logout olarak dusunmemek lazim. Esasinda arka planda bir cok islem yapiliyor ve kullanicinin login olmasinin yaninda kullanicinin spesifik ozellikleri belirleniyor. Dolayisiyla view'lerimizde herhangi bir kod calistirilmadan once authentication islemi arka planda tamamlaniyor. Permissions da dahil olmak uzere diger tum islemler bu authentication islemleri bitiminde calismaya basliyor.
- Authentication tek basina gelen isteklerin izinlendirilmesini yani engellenmesini yapmiyor. Sadece gelen istek icerisinde yer alan credentials bilgisini islemlendiriyor.
- DRF bize hazir birkac farkli authentication sistemi/yolu sunuyor. Bu dersimizde en onemli olanlari, bunlari avantaj ve dezavantajlarini, en onemlisi ise ne zaman ve nerede kullanilmasinin daha uygun olacagini inceleyecegiz.
- Ayrica yeni bir Authentication standardi olan JWT konusuna bakarak baziucuncu parti paketler kullanarak bu yeni standardin nasil uygulanabilecegini gorecegiz.

## Basic Authentication

En ilkel ve en az guvenli authentication sistemi. Sadece tst amacli yani gelistirme asamasinda kullanilmasi uygundur. Islemler su siralama ile kabaca ozetlenebilir.

1. Kullanici (ya da istemci) sunucuya istek gonderir.
2. Sunucu, kullaniciya HTTP 401 Unauthorized cevabini gonderir. Ancak bu cevabin icerisinde *WWW-Authenticate* header(baslik)'ini gonterir. (WWW-Authenticate:Basic gibi)
3. Kullanici bu baslik bilgisine gore ilgili kimlik bilgilerini BASE64 formatinda sunucuya gonderir ama bu trafikte tum kimlik bilgileri **SIFRELENDIRILMEDEN** gonderilir.
4. Sunucu gelen kimlik bilgilerini kontrol eder ve cevap olarak 200 ya da 403 durum kodunu kullaniciya gonderir.
5. Yani eger *https* protokolu kullaniliyorsa, agda yer alan herkes bu kimlik bilgilerini goruntuleyebilir. Kisacasi hacklenirsiniz.
6. Bir Session management olayina giremezsiniz cunku bilgiler aciktadir. Yani her defasinda login yapmak icin bilgileri tekrar almaniz gerekir.


## Token Authentication

Mobil istemciler ya da desktop islemciler icin en uygun ve ideal sistemdir. Request/Response dongusu asagidaki sekilde ozetlenebilir:
1. istemci, kimlik bilgilerini bir defa sunucuya gonderir
2. Sunucu kimlik bilgilerini kontrol eder, eger gecerli ise string ifade formatinda bir token(sifre) olusturur ve istemciye(kullanici) bu token'i gonderir.
3. Istemci her istek yaptiginda Authorization Header icerisinde bu token'i gonderir ve bu sekilde authentication islemini cozer
4. Her istek yapildiginda, sunucu bu token'i kontrol eder ve gecerliyse diger islemlere izin verir.

Biz derslerimizde Token Authentication uzerinden gidecegiz. Requests modulunu de kullanarak kendi yazacagimiz scriptlerle bu authentication surecini yonetecegiz.

Token Authentication, React ve Vue gibi JS frameworkleri ile yazilan "Single Page Application" (tek sayfa uygulamalar)'da en cok kullanilan yetkilendirme sistemi. Bu kullanimlarda tenel olarak bu token cookies icerisine ya da local storage (yerel depolama) icerisine kaydedilir. Ancak, altini cizerek soylemekte fayda var. Local storage icierisinde token kaydetmek bircok kolaylik saglasa da JS kodlari ile erisime acik oldugu icin **XSS saldirilarina** maruz kalma riskini son derece arttirir. Bu yuzden *httpOnlyCookie* icerisine kaydetme islemi, JS ile erisim olmadigi icin cok daha guvenli bir sistemdir. Ancak local storage'a gore daha az esnek kullanim sunar/

Bu arada tum gelistirme senaryolari icin gecerli global olarak kabul edilmis bir kullanimdan bahsedemeyiz cunku her projenin kendine gore gereksinimleri degismektedir. Dolayisiyla bizim yapmamiz gereken, mumkun olan en yuksek guvenlik ve uygun cozumu uygulamaktir. Bizim projelerimiz gibi projelerde DRF resmi dokumantasyonu *Session Authentication* kullanimini tavsiye etmektedir. Aslinda projelerimizde birden fazla authentication sistemi kullanabiliriz.

## DRF - Session Authentication
Bu authentication sistemi Django'nun default authentication sistemini kullanmaktadir. AJAX istemciler ile yurutulecek bir web aplikasyonu icin en uygun ve en guvenilir sistem diyebiliriz. Session ve Cookie kombinasyonlarini kullanmaktadir.

### Nasil Calisir?
Request/Response dongusu asagidaki sekilde ozetlenebilir:
1. Istemci, genel olarak bis web sayfasinda form doldurarak kimlik bilgilerini sunucuya gonderir.
2. Sunucu bilgileri kontrol eder ve dogrulanirsa bir Session Object (Oturum Nesnesi) olusturarak veri tabanina kaydeder ve Session ID'sini istemciye gonderir.
3. Bu session id browser icerisinde bir cookie'de kaydedilir. Istemci her istek yaptiginda bu session id'sini de sunucuya gonderir ve her istekte bu id kontrol edilir.
4. Istemci logout oldugunda bu session id her iki tarafta da silinir. Tekrar login oldugunda yeni session id olusturulur.

Yukaridaki dongude gerceklestirilen login islemi basarili olursa Django bize `request.user` ile erisebilecegimiz bir user nesnesi gonderecektir, ki bunu zaten daha once kullandiginizi dusunuyorum. Login olmayan kullanicilar icin ise AnonymousUser nesnesi donmektedir. Daha once permissions konusunu islerken bununla karsilasmistik.

    ONEMLI: Authentication gerceklestestikten sonra herhangi bir unsafe request yapildiginda (POST, PUT, PATCH, DELETE) frameworkumuz bizden bir CSRF token de gondermemizi isteyecektir. Yani bu requestlerde CSRF token'i da dahil etmemiz gerekecektir.

## JSON Web TOkens (JWT)

JWT daha yeni bir standart. Yapisi sebebiyle diger token sisteminden farkli olarak veri tabani dogrulamasi istememektedir. DRF ile olusturulan REST API servislerinde `django-rest-framework-simplejwt` paketi yuklenerek cok kolay bir sekilde uygulanabilir. Bu uygulamayi dersimizde islemeyi planlamiyoruz ancak bu seszonu tamamladigimizda rahatlikla kullanabilecek seviyeye geleceksiniz.

# Django Rest Auth - Bolum 1
Bu ve onumuzdeki dersimizde django-rest-auth paketini kullanarak endpointlerimiz uzerinde kayit ve authentication uygulamalarini yapmayi ogrenecegiz. iOS ve Android uygulamalari, kolaylikla backendimizle REST uzerinden iletisime gecip, uygulamalarimizin sagladigi hizmetleri kolaylikla kullanabilecekler.

[Authentication Django REST Framework](https://www.django-rest-framework.org/api-guide/authentication/)
REST dokumanini inceleyecek olursak ayni pagination ve permission konularinda oldugu gibi *settings.py* dosyasina ilgili satirlari ekleyerek authentication icin global policy olusturabiliriz. 

*core/settings.py*
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

Biz, projemizin *settings.py* dosyasinda BasicAuthentication yerine, TokenAuthentication'i ekledik. Ancak bir onceki derste konustugumuz gibi bu tokenler ve sessionlar veritabanina kaydedilecegi icin, bizim `INSTALLED_APPS` icerisine `rest_framework_authtoken` kaydini yapmamiz ve migrationlari olusturmamiz lazim.

*core/settings.py*
```python
...
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django_extensions',
    'rest_framework',
    'rest_framework.authtoken',
    'rest_auth', # pip install django-rest-auth
    'profiller.apps.ProfillerConfig,

...
```

Migration'lari olusturduktan sonra server'imizi calistirip admin panelinde `AUTH TOKEN` uygulamasi altinda `Tokens` modelini gorebiliriz. Burada manual olarak kullanicilarimiza token verebiliriz ama tabi ki DRF bu islemleri arka planda bizim icin otomatik olarak gerceklestirecek. Bunun icin cok kullanisli olan bir paket yukleyecegiz. Bu paket bize registration ve login/logout icin gerekli endpointleri otokatik olarak verecek.

```terminal
> pip install django-rest-auth requests
```

Endpoint scriptlerimizi yazabilmemiz icin request kutuphanesini de yukledik. 

Simdi registration ve login/logout islemleri icinurl'leri tanimlamamiz lazim. 
*core/urls.py*
```python
from django.contrib import admin
from django.url import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api-auth/", include("rest_farmework.urls")),
    path("api/rest-auth/", include("rest_auth.urls")),
]

from django.conf import settings
from django.conf.urls.static import static

if settings.DEBUG == True:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### Clientlarimiz (Istemcilerimiz)
Amacimiz, sunucumuza istek gonderip kayit ve login islemlerini REST API uzerinden yapmak oldugu icin kendi client'imizi yazalim. bunun icin core ve proje klasorlerinin bulundugu dizinde *clients* adinda bir klasor olusturalim. Kendi sunucumuza istek gonderelim.

*clients/token_auth_test1.py*
```python
import requests
from pprint import pprint

def client():
    credentials = {
        "username": "testuser",
        "password": "testuser321.."
    }

    response = requests.post(
        url = "http://127.0.0.1:8000/api/rest-auth/login/"
        data = credentials        
    )

    print(f"Response Status Code: {response.status_code}")
    response_data = response.json()
    pprint(response_data)

if __name__ == "__main__":
    client()
```

```terminal
> python token_auth_test1.py
Response Status Code: 200
{'key': '8t6612nei418198srq83d86akb8az'}
```

Boylece basarili sekilde login yapmis olduk. Artik (sunucumuz tarafindan veritabanina da kaydedilmis olan) bu token'i kullanarak unsafe requestler yapip, sunucumuz dahilindeki endpointlere erisebiliriz. Eger admin sayfasina giris yaparsak `admin/Auth Tokens/Tokens` modeli altinda ilgili keyi goruntuleyebilirsiniz.

### Ornek Bir View Uzerinden Authentication Sistemimizi Test Edelim

*profiller/api/views.py*
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated
from profiller.models import Profil
from profiller.api.serializers import ProfilSerializer

class ProfilList(generics.ListAPIView):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated,)
```

*profiller/api/urls.py*
```python
from django.urls import path
from profiller.api import views

urlpatterns = [
    path("kullanici-profilleri", views.ProfilList.as_view(), name="profiller"),
]
```

*core/urls.py*
```python
...
urlpatterns = [
    ...
    path("api/", include("profiller.api.urls")),
]
...
```

*clients/token_auth_test2.py*
```python
import requests
from pprint import pprint

def client():
    response = requests.get(
        url = "http://127.0.0.1:8000/api/rest-auth/login/"       
    )

    print(f"Status Code: {response.status_code}")
    response_data = response.json()
    pprint(response_data)

if __name__ == "__main__":
    client()
```

```terminal
> python token_auth_test2.py
Status Code: 401
{'detail': 'Authentication credentials were not provided.'}
```

*clients/token_auth_test3.py*
```python
import requests
from pprint import pprint

def client():
    token = "Token 8t6612nei418198srq83d86akb8az"

    headers = {
        "Authorization": token,
    }
    response = requests.get(
        url = "http://127.0.0.1:8000/api/rest-auth/login/",
        headers = headers,
    )

    print(f"Status Code: {response.status_code}")
    response_data = response.json()
    pprint(response_data)

if __name__ == "__main__":
    client()
```

```terminal
> python token_auth_test2.py
Status Code: 200
[{'bio': 'Kurucu',
  'foto': 'http://127.0.0.1:8000/media/profil_fotolari/2020/10/OI4FKU0.jpg',
  'id': 1,
  'sehir': 'Ankara',
  'user', 'alperakbas'  
}]
```

# Django Rest Auth - Bolum 2 - Registration

django-allauth kutuphanesi dokumantasyonu: [Installation - django-allauth](https://django-allauth.readthedocs.io/en/latest/installation.html)

Olusturdugumuz REST API mimarisi ile kulanici kaydi da kabul edebilmemiz gerekiyor. Bunun icin biz hali hazirda varolan ve baya kuvvetli olan bir ucuncu parti python paketini kullancagiz. Boylelikle registration icin end-pointler olusturmamiza gerek kalmayacak, daha da onemlisi kayit islemleri icin gerekli tum islemleri arka planda halletmis olacagiz. Biz tabiki dersimizi sade ve anlasilabili kilabilmek maksadiyla basit bir ornek yapacagiz. Ancak django-allauth kutuphanesini kullanacakporjelerimize social login (yani facebook, google vb.) ozelligi de ekleyebiliriz. Sunu da blertmekte fayda var, django-allauth kutuphanesi restfarmework'e ozgu degil, klasik django uygulamalarinda da cok kullanilan bir kutuphane. Isleri de ciddi anlamda kolaylastirmakta.

## 3.7.1 Ayarlamalar

Ilk olarak python sanal ortamimizda `pip install django-allauth` komutu ile ilgili kutuphaneyi yuklememiz gerekiyor. Yukleme islemini takiben *settings.py* dosyasinda, INSTALLED_APPS kismina ilgili kayitlari yapmamiz gerekiyor. Sununda altini cizmemiz lazim, django-allauth kutuphanesi ile django-rest-auth kutuphanesi entegre bir sekilde calisiyor, yani birini kullanabilmemiz icin digerini de kullanmamiz gerekmekte. Bu ders notlarinin olusturuldugu tarih itibariyle.

```terminal
> pip install django-allauth
```

*core/settings.py*
```python
...

INSTALLED_APPS = [
    ...
    'rest_auth',
    ### registration end-pointlerimiz icin
    'allauth',
    'allauth.account',
    'allauth.socialaccount',    # social login icin gerekli, derslerimizde gormeyecegiz
    'rest_auth.registration',   # bunu simdi yapiyoruz cunku rest_auth, bu uygulama icin all_auth'u kullaniyor
    'django.contrib.sites',     # django ile gelen uygulama, bunu da kayit etmemiz gerekiyor
    ###
    'profiller.apps.ProfillerConfig,

...
```

*settings.py* dosyasiyla ilgili islemlerimiz hala bitmedi, *settings.py* dosyasinin sonuna asagidaki degisiklikleri de eklememiz gerekiyor.

*core/settings.py*
```python
SITE_ID = 1

ACCOUNT_EMAIL_VERIFICATION = 'none' # kayit esnasinda email onayi istiyor muyuz?
ACCOUNT_EMAIL_REQUIRED = (True,)    # kayit esnasinda kullanici email adresi vermeli mi?
```

Veri tabanimiza yeni tablolar da eklenecegi icin migrations'larimizi yapmamiz gerekiyor.
```terminal
> python manage.py makemigrations
> python manage.py migrate
```

Migrate komutunu da calistirdiktan sonra sunucumuzu calistirirsak admin sayfasinda asagidaki yeni sayfalari (uygulama) ve modelleri gormemiz gerekiyor.

- Accounts
  - Email Addresses
- Sites
  - Sites
- Social Accounts
  - Social accounts
  - Social application tokens
  - Social applications

Biz kayit islemlerini API end-pointler uzerinden yapacagimiz icin esasinda Sites, Social Account ile ilgilenmiyoruz. Ancak klasik django projelerinde bu yapilari kullanarak social login islemleri de yapabilirsiniz. Bunun icin sosyal medya uygulamasinda (or. Facebook) da gelistirici sayfalarina giderek bazi ek islemler yapmamiz gerekecek fakat dersimizin konusu degil.

## 3.7.2 URL'lerimiz

Simdi, end-point'lerimizi tam olarak bitirebilmek icin ana *urls.py* dosyamizda bazi eklemeler yapmamiz gerekecek.

*core/urls.py*
```python
...
urlpatterns = [
    ...
    path("api/rest-auth/registration/", include("rest_auth.registration.urls")),
]
...
```

*clients/registration_test.py*
```python
import requests
from pprint import pprint

def client():
    credentials = {
        "username": "rest_test_user",
        "email": "test@test.co",
        "password1": "testuser321..",
        "password2": "testuser321..",
    }

    response = requests.post(
        url = "http://127.0.0.1:8000/api/rest-auth/registration/"
        data = credentials        
    )

    print(f"Response Status Code: {response.status_code}")
    response_data = response.json()
    pprint(response_data)

if __name__ == "__main__":
    client()
```

```terminal
> python registration_test.py
Response Status Code: 201
{'key': '35ne4nei41u8198srq83d86akb8az'}
```

Ayni dosyanin ikinci kez calistirilmasi durumunda ise:
```terminal
> python registration_test.py
Response Status Code: 400
{'email': ['A user is already registered with this e-mail address.']}
```

# 3.8 ViewSets ve Routers - Part 1

Simdiye kadarki derslerimizde farkli stratejelerle cesitli viewler yazdik. APIView sinifindan gelerek ve bazi mixinler kullanarak generic viewlere kadar bircok farkli yol izledik. Her yeni konumuzda bircok proglamlama senaryosunda hemen hemen ayni islemler benzer mantikla yapilmakta dedik. Bu islemler neydi:
- Listeleme
- Detayli goruntuleme
- Olusturma
- Guncelleme
- Silme

Ornegin ikinci sezonda isledigimiz concret viewlerde bile (ki suana kadar gordugumuz en ust soyutlama ve en az kodlama gerektiren seviyeydi), listeleme, detayli goruntuleme, silme gibi islemler icin farkli viewler yazmamiz gerekti. Ikinci sezonda generic viewleri islerken yazdigimiz ilk basit concrete viewler asagidaki gibiydi:

*kitaplar/api/views.py*
```python
from rest_framework.generics import GenericAPIView
from rest_framework.mixins import ListModelMixin, CreateModelMixin
from rest_framework import generics
from kitaplar.api.serializers import KitapSerializer, YorumSerializer
from kitaplar.models import Kitap



class KitapListCreateAPIView(generics.ListCreateAPIView):
    queryset = Kitap.objects.all()
    serializer_class = KitapSerializer


class KitapDetailAPIView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Kitap.objects.all()
    serializer_class = KitapSerializer
```

Dedigimiz gibi benzer islemler benzer mantik siralamasinda yapildigi icin, isleri daha da hizlandirmak ve kolaylastirmak mantigindan hareketle DRF'i insaa eden gelistiriciler, ViewSet denilen bir konsepti kullanimimiza acmislar. Aslinda ViewSet, View kumeleri seklinde tercume edilebilir. Yani ozel olarak yukaridaki gibi birkac view'de yapmamiz gereken islemleri, tek bir view icerisinde halledebilmemiz icin gerekli bir konsept. Gelistirici arkadaslar bu kadariyla da yetinmemis, `routers` denilen konseptlerle de devamli olarak kullanmakzorunda oldugumuz URL'leri de otomasyon baglamislar. Yani en iyi yazilimci, tembel yazilimci diyebiliriz belki de.

Ozet olarak, ornegin ViewSet'ler ile hem bir sorgu kumesindeki elemanlari listeleyebiliriz, hem de bu kumeden ya da ilgili modelden tek bir nesneyi goruntuleyebiliriz. ViewSet'ler de diger bir cesit ClassBasedView. Yalniz `.get()`, `.post()` gibi metodlar yerine `.list()`, `.create()` gibi aksiyona ve sonuca yonelik metodlar icermekteler. Zaten her zaman yaptigimiz gibi dersimizde kaynak kodlarina bir goz atacagiz. 

Yukarida da bahsettigimiz gibi, ViewSet'ler genel olarak `router`'lar ile kullanilmaktalar. Boylelikle genel kullanimda olmasi gereken aksiyonlara yonelik uygun url path konfigurasyonlarini da otomatik olarak olusturmus oluruz.

En son asagida verilen, profilleri listeledigimiz view'imizi yazmistik

*profiller/api/views.py*
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated
from profiller.models import Profil
from profiller.api.serializers import ProfilSerializer

class ProfilList(generics.ListAPIView):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated,)
```

Concrete view'leri isledigimiz derslerimizde, bu viewlerin ne kadar guclu oldugunu zaten gormustuk. Yukaridaki view'imiz ile listeleme ve olusturma islemlerini hallettik, ancak detayli goruntuleme yani tek bir kullanici profili goruntuleme islemi icin normlade yukaridakine benzer bir tane daha view (`generics.RetrieveUpdateDestroyAPIView` ya da `generics.RetrieveAPIView` sinifindan tureterek) yazmamiz gerekiyor. Ite burada ViewSet'ler devreye giriyor. ViewSet'ler ile birbiriyle alakali bir cok islemi tek bir view'de toparlama sansimiz var. Boylelikle hem kod yukumuz azaliyor, hem de kodumuz daha okunabilir ve temiz olmus oluyor.

Simdi yukarida yazdigimiz ProfilList view'imizi biraz degistirerek bu view'imize detail yani detayli goruntuleme yetenegini de kazandiralim.

*profiller/api/views.py*
```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated
from profiller.models import Profil
from profiller.api.serializers import ProfilSerializer
from rest_framework.viewsets import ReadOnlyModelViewSet

class ProfilViewSet(ReadOnlyModelViewSet):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated,)
```

Yukaridaki degisiklikleri yaparak tek bie view'e hem listeleme hem de detayli goruntuleme yetenegi kazandirmis olduk. Oncelikle `rest_framework.viewsets` yapisindan `ReadOnlyModelViewSet` sinifini cektik ve view'imizi bu siniftan turettik. View'imizin de ismini degistirdik. Baska herhangi bir degisiklik yapmadik. Boylelikle kodumuz cok daha kisa, okunabilir ve duzenli bir hale gelmis oldu. Bir cok kodu bedavaya getirmis olsak da `urls.py` dosyasinda bir dizi degisiklik yapmamiz gerekecek.

*profiller/api/urls.py*
```python
from django.urls import path
from profiller.api.views import ProfilViewSet
# Hala, detayli goruntuleme ve listeleme icin iki ayri endpoint olusturmamiz gerekiyor.
profil_list = ProfilViewSet.as_view({"get": "list"})
profil_detay = ProfilViewSet.as_view({"get": "retrieve"})

urlpatterns = [
    path("kullanici-profilleri", profil_list, name="profiller"),
    path("kullanici-profilleri/<int:pk>", profil_detay, name="profil-detay"),
]
```

Ancak dersimizin girisinde de bahsettigimiz uzere, ViewSet yapisiyla birlikte gelen `router`yapisi, klasik bir sekilde hazirlamamiz gereken endpoint'lerimizi otomatik olarak hazirlamamizi saglamakta. Simdi routers yapisi ile url endpoint'lerimizi nasil otomatik olarak olusturabilecegimize bakalim.

*profiller/api/urls.py*
```python
from django.urls import path, include
from profiller.api.views import ProfilViewSet
from rest_farmework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'profiller', ProfilViewSet)

print("_________________")
print(router.urls)
urlpatterns = [
    path("", include(router.urls)),
]
```
Yukaridaki dosyada, yani urls.py dosyamizda, DefaultRouter sinifini kullanarak, bir router nesnesi olusturduk ve ProfilViewSet'imizi, profiller uzantisi ile nesnemiz icerisinde kaydettik. urlpatterns liste nesnesi icerisinde de router.urls yapisini dahil ettik. Boylece `http://127.0.0.1:8000/api/profiller` ve `http://127.0.0.1:8000/api/profiller/<int:pk>/` endpoint'lerini otomatik olarak olusturmus olduk. Hatta artik `http://127.0.0.1:8000/api/` adres ile API Root'umuz icin de bir sayfa `/` json api beslemesi olusturmus olduk.

# 3.9 ViewSets ve Routers - Part 3
Tabi biz `rest_framework.viewsets.ReadOnlyModelViewSet` yapisi uzersinden view sinifi  olusturdugumuz icin sadece bilgi goruntulume islemi yapabiliyoruz. Peki bu view'imiz icerisindeki queryset'e ait herhangi bir nesneyi update etmek istersek ne yapacagiz. Simdi ProfilViewSet'imizi buna gore tekrar duzenleyelim.

Ancak burada dikkat etmemiz gereken birkac mantiksal konu var. Ornegin, biz bu ProfilViewSet sinifimiza create yani olusturma yetkisi vermeyecegiz, cunku olusturma islemini biz onceki videolarimizda, api uzerinden `rest_auth.registration` ile yapmistik. Dolayisiyla bizim update, list, retriew yetkilerini vermemiz lazim. Bunun icin mixin'leri kullanacagiz. Ayrica, permissions konusunda da gordugumuz gibi, bir permission (izin) sinifi da ekleyerek, sadece profil sahibinin kendi profilini guncelleyebilmesini saglamamiz lazim. Oncelikle profiller/api klasoru icerisinde *permissions.py* dosyasini olusturalim ve izin sinifimizi yazalim. 

*profiller/api/permissions.py*
```python
from rest_framework import permissions

class KendiProfiliYaDaReadOnly(permissions.BasePermission):

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        
        return obj.user ==  request.user
    
```

Artik view'imizi guncellemeye haziriz:

*profiller/api/views.py*
```python
from rest_framework.permissions import IsAuthenticated
from profiller.models import Profil
from profiller.api.serializers import ProfilSerializer
from profiller.api.permissions import KendiProfiliYaDaReadOnly
from rest_framework.viewsets import GenericViewSet
from rest_framework import mixins

class ProfilViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    GenericViewSet,
):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated, KendiProfiliYaDaReadOnly)
```


# 3.10 ViewSets ve Routers - Part 3

Bu derste, profil durum mesajlarinin yayimlanabilmesi icin bir view yazacagiz. ModelViewSet kullanarak cok daha az kod ile durum mesaji olusturma islemini yapacagiz. Ancak, burada dikkat etmemiz gereken ince bir mantiksal nokta var:

*profiller/models.py*
```python
class ProfilDurum(models.Model):
    user_profil = models.ForeignKey(Profil, on_delete=models.CASCADE)
    durum_mesaji = models.CharField(max_length=240)
    yaratilma_zamani = models.DateTimeField(auto_now_add=True)
    guncellenme_zamani = models.DateTimeField(auto_now=True)

    class Meta:
        verbose_name_plural = "DurumMesajlari"
    
    def __str__(self):
        return str(self.user_profil)
```

ProfilDurum modelimizi hatirlayalim. ProfilDurum modelimizden bir nesne olusturdugumuz zaman mutlaka `user_profil` altinda bir profil nesnesi vermemiz lazim. Bu sebeple onceki derslerimizde de gordugumuz gibi, `perform_create()` metodunu override etmemiz gerekecek. Ayrica, kullanicilarin sadece kendi durum mesajlarini guncelleyip, silebilmesi icin de bir permission sinifi daha yazmamiz gerekecek. Zaten *permissions.py* dosyamizi olusturmustuk. Oncelikle permission sinifimizi yazalim.

*profiller/api/permissions.py*
```python
...

class MesajSahibiYaDaReadOnly(permissions.BasePermission):

    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        
        return obj.user_profil ==  request.user.profil
    
```

View'imizin son hali asagidaki gibi olmali:

*profiller/api/views.py*
```python
from rest_framework.permissions import IsAuthenticated
from profiller.models import Profil
from profiller.api.serializers import ProfilSerializer, ProfilDurumSerializer
from profiller.api.permissions import KendiProfiliYaDaReadOnly, MesajSahibiYaDaReadOnly
from rest_framework.viewsets import GenericViewSet, ModelViewSet
from rest_framework import mixins

class ProfilViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    GenericViewSet,
):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated, KendiProfiliYaDaReadOnly)


class ProfilDurumViewSet(ModelViewSet):
    queryset = ProfilDurum.objects.all()
    serializer_class = ProfilDurumSerializer
    permission_classes = (IsAuthenticated, MesajSahibiYaDaReadOnly)

    def perform_create(self, serializer):
        user_profil = self.request.user.profil
        serializer.save(user_profil=user_profil)
```

Yukarida goruldugu uzere ilgili `user_profil` alanini `perform_create()` metodunu override ederek cozduk.

Eger `perform_create()` metodunda nasil bir islem yapildigini hatirlamiyorsak, yapmamiz gereken aslinda `ModelViewSet` kaynak kodunda, oradan da `mixins.CreateModelMixin` yapisinin koduna bakmak.

Bu islemlerin ardindan, `urls.py` icerisinde daha onceden olusturdugumuz `router`'imiza bu yeni view'imizi kaydetmemiz gerekiyor. Boylelikle, endpoint'lerimizi de otomatik olarak olusmus olacak.

*profiller/api/urls.py*
```python
from django.urls import path, include
from profiller.api.views import ProfilViewSet, ProfilDurumViewSet
from rest_farmework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'profiller', ProfilViewSet)
router.register(r'durum', ProfilDurumViewSet)

urlpatterns = [
    path("", include(router.urls)),
]
```

Artik API Root'umuza (http://127.0.0.1:8000/api/) gittigimizde asagidaki gibi bir goruntu almamiz lazim:

```json
{
    "profiller": "http://127.0.0.1:8000/api/profiller/",
    "durum": "http://127.0.0.1:8000/api/durum/"
}
```

Goruldugu gibi durum olarak kaydettigimiz yeni router nesnemiz API Root'umuzda da goruluyor. Artik http://127.0.0.1:8000/api/durum/ adresine istek yaptigimizda, ModelViewSet ile olusturdugumuz aksiyonlari kullanabiliriz.

`ViewSet` ve `router` yapilari oldukca kuvvetli yapilar. Ancak, en ust soyutlama seviyesindeki yapilar. Dolayisiyla bazi aksiyonlar icin kaynak kodunu ve hatta ilgili dokumantasyonu incelememizde fayda var.

# 3.11 ViewSets ve Routers - Part 4

Bir cok gercek projede ise ViewSet'ler ve Concrete viewler birlikte kullanilmakda. Hatirlarsaniz bizim projemizde, profil fotosnunu guncellenmesi icin ayri bir serializer yazmistik. Simdi, kullanicilarin profil fotografini guncelleyebilmesi icin yeni bir view yazalim.

*profiller/api/views.py*
```python
from rest_framewok import generics
from profiller.api.serializers import ProfilSerializer, ProfilDurumSerializer, ProfilFotoSerializer

...

class ProfilFotoUpdateView(generics.UpdateAPIView):
    # serializer'imizda sadece foto alanini dahil etmistik
    # bu sebele bir update aksiyonunda sadece profil alani etkilenecek
    serializer_class = ProfilFotoSerializer
    permission_class = (IsAuthenticated,)
    # izin yazmamiza gerek yok, cunku get_object ile kisitladik

    def get_object(self):
        # bize tek bir profil nesnesi lazim
        # bu sebele queryset belirlemiyoruz ve GenericAPIView ile
        # gelen get_object metodunu override ediyoruz
        profil_nesnesi = self.request.user.profil
        return profil_nesnesi
```

Yukarida *profiller/api/views.py* dosyasinin sadece ilgili kismi verilmistir. Daha onceden yazdigimiz ProfilFotoSerializer'imiz ile zaten sadece foto alanini dahil ettik. Bu nedenle, update aksiyonumuz sadece o alani yani foto alanini etkileyecek.

Peki neden queryset belirlemedik? Cunku, biz tek bir profil nesnesi ile ilgileniyoruz. Bizim alacagimiz profil nesnesinin login olmus kullanicinin profili olmasi lazim. Biz bu islemi GenericAPIView ile gele `get_object()` metodunu icerisinde cozuyoruz. Bu sebeple hem queryset vermemize hem de fazladan permission yazmamiza gerek kalmadi.

Artik *urls.py* dosyamiz icerisinde bir endpoint yazmamiz lazim. Ancak biz bu view'de ViewSet kullanmadik. Dolayisiyla `router`'a kayit yapamayiz. *profiller/api/urls.py* dosyamiz asagidaki gibi olmali:

*profiller/api/urls.py*
```python
from django.urls import path, include
from profiller.api.views import ProfilViewSet, ProfilDurumViewSet, ProfilFotoUpdateView
from rest_farmework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'profiller', ProfilViewSet)
router.register(r'durum', ProfilDurumViewSet)

urlpatterns = [
    path("", include(router.urls)),
    path("profil_foto/", ProfilFotoUpdateView.as_view(), name="foto-update"),
]
```

Simdi http://127.0.0.1:8000/api/profil_foto/ adresine istek yaparsak login olmus kullanicinin profil fotografini degistirebiliriz.  

# 3.12 DRF'de Filtreleme

Su ana kadar kullandigimiz view'lerimizde, hep tam bir sorgu kumesi (queryset) kullandik. Ancak bazi durumlarda, hatta cogu durumda bir sorgu kumesindeki belirli bir ya da birden fazla ogeyi cekmek isteyebiliriz. Bunu da onceden belirlenmis bazi kriterlere baglamak isteyebiliriz.

Bu dersimizde, `get_queryset()` metodunu ve DRF filtreleme mantigini kullanarak ve kisisellestirerek, REST API'lerimizi daha da gelistirecegiz.

ProfilDurumViewSet sinifimizi modifiye ederek baslayalim. Hikayemizde, durum endpoint'imize bir username ekleyerek istenen kullanicinin durum mesajini/mesajlarini goruntuleyebilmek olsun.

*profiller/api/views.py* `ProfilDurumViewSet`'imiz asagidaki gibi olmali:
```python
class ProfilDurumViewSet(ModelViewSet):
    serializer_class = ProfilDurumSerializer
    permission_classes = (IsAuthenticated, MesajSahibiYaDaReadOnly)

    def get_queryset(self):
        queryset = ProfilDurum.objects.all() # filtreleme yapmak icin queryset burada aliyoruz
        username = self.request.query_params.get("username") # url'imizde belirleyecegimiz username parametresi

        if username:
            queryset = queryset.filter(user_profil__user__username=username)
        
        return queryset

    def perform_create(self, serializer):
        user_profil = self.request.user.profil
        serializer.save(user_profil=user_profil)
```

*profiller/api/urls.py*
```python
from django.urls import path, include
from profiller.api.views import ProfilViewSet, ProfilDurumViewSet, ProfilFotoUpdateView
from rest_farmework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'profiller', ProfilViewSet)
router.register(r'durum', ProfilDurumViewSet, basename="durum")

urlpatterns = [
    path("", include(router.urls)),
    path("profil_foto/", ProfilFotoUpdateView.as_view(), name="foto-update"),
]
```

Yukaridaki gibi `basename`'i acik bir sekilde belirtmez isek "`AssertionError: 'basename' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset'` attribute" hatasini alacagiz. Cunku, viewset'imizin islemi tamamlamasi icin bir queryset degiskenine ihtiyacimiz var ve biz queryset'i dolayli yoldan, yani `get_queryset()` metodu ile olusturuyoruz.

Boylelikle http://127.0.0.1:8000/api/durum/?username=deneme_kullanicisi seklinde bir url ile deneme_kullanicisi'na ait tum durum mesajlarini goruntuleyebiliriz.

Peki model uzerinde, ornegin http://127.0.0.1:8000/api/profiller/?search=izmir seklinde bir url vasitasi ile ilgili modele ait istenilen alanlarda arama/filtreleme yapmak isteksek? Simdi bu islemi, ProfilViewSet sinifimiz uzerinden filter_backends ve DRF ile hazirda gelen SearchFilter yapisini kullanarak yapalim.

*profiller/api/views.py*
```python
...
from rest_framework.filters import SearchFilter

class ProfilViewSet(
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    GenericViewSet,
):
    queryset = Profil.objects.all()
    serializer_class = ProfilSerializer
    permission_classes = (IsAuthenticated, KendiProfiliYaDaReadOnly)
    filter_backends = (SearchFilter,)  # birden fazla arama filtresi dahil edilebilir
    search_fields = ("sehir",)  # profil modeline ait arama yapilmasi istenen field'lar
```

Yukaridaki islemleri, dogru bir sekilde yaparsak artik http://127.0.0.1:8000/api/profiller/?search=Ankara benzeri endpoint'ler uzerinden arama/filtreleme islemlerini gerceklestirebiliriz. Eger, browsable api sayfamiza dikkat ederseniz, artik html ara yuzumuzde *Options* butonu yaninda bir *Filter* butonumuz, dolayisiyla filtreleme icin form yapimiz otomatik olarak olustu. Ayrica `?search=ank` gibi parcali aramalari da desteklemekte.

    SearhFilter'a ait diger arama yontemleri icin [DRF SearchFilter](https://www.django-rest-framework.org/api-guide/filtering/#searchfilter)  

Burada en onemli nokta, DRF ile farkli filter_backend'ler geldigi (iki adet) ve hali hazirda varolan bazi cok guclu paketlerle yeni filtreleme backend'lerini kolaylikla ekleyebilecegimizi bilmemiz. Ayrica, *settings.py* dosyasinda REST_FRAMEWORK ayarlari icerisine DEFAULT_FILTER_BACKENDS ekleyerek default filtreleme seceneklerini de belirleyebiliriz. Ancak en maktikli kullanim sekli view bazinda filtreleme belirlemek. Boylece programimizin davranislarini daha iyi kontrol altinda tutabiliriz.
