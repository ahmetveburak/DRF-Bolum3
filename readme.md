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