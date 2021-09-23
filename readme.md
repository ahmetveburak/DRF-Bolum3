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

*settings.py*
```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

Biz, projemizin *settings.py* dosyasinda BasicAuthentication yerine, TokenAuthentication'i ekledik. Ancak bir onceki derste konustugumuz gibi bu tokenler ve sessionlar veritabanina kaydedilecegi icin, bizim `INSTALLED_APPS` icerisine `rest_framework_authtoken` kaydini yapmamiz ve migrationlari olusturmamiz lazim.

```python
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

Boylece basarili sekilde login yapmis olduk. Artik (sunucumuz tarafindan veritabanina da kaydedilmis olan) bu token'i kullanara unsafe requestler yapip, sunucumuz dahilindeki endpointlere erisebiliriz. Eger admin sayfasina giris yaparsak `admin/Auth Tokens/Tokens` modeli altinda ilgili keyi goruntuleyebilirsiniz.

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