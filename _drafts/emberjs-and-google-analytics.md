---
layout: post
title: Ember.js ve Google Analytics
---

## Ember.js ve Google Analytics
---
Uygulamanıza [Google Analytics](http://www.google.com/analytics/) eklemek istiyorsanız Ember.js'ın resmi sitesinde önerilen [yöntemi](http://emberjs.com/guides/cookbook/helpers_and_components/adding_google_analytics_tracking) uygulamanız yeterli olacaktır. Biraz daha derine indiğiniz zaman şöyle bir ihtiyaç doğduğunu fark edeceksiniz yerel bilgisayarınızda çalışırken google analitik kodunu uygulamanın içine enjekte etmemize gerek yok. Hatta bu durum sizin veya pazarlama bölümünün yapacağı analizlerin daha temiz olması için önemli bir hal almaya başlayacak.

###Initializers
---
Ember.js uygulama başlamadan önce farklı bileşenleri barından `container` objesine `dependency injection` uygulayabileceğimiz veya uygulama yapısını değiştirmemize olanak sağlayan [`Initializer`](http://emberjs.com/api/classes/Ember.Application.html#toc_initializers) yapısını bu blog'da Google Analytics için kullanacağız.

>Bu ve diğer bütün blog yazılarımda [ember-cli](https://github.com/stefanpenner/ember-cli/) kullandığınızı varsayarak ilerleyeceğim.

uygulama dosya hiyerarşisinde **initializers** klasörü altına **analytics.js** dosyası oluşturacağız. ember-cli'ın bize sağladığı [generator](http://www.ember-cli.com/#generators-and-blueprints) ile initializer oluşturmak için: `ember g initializer analytics` kodunu terminalde çalıştırmamız yeterli olacaktır.

```javascript
import ENV from 'app/config/environment';

export var initialize = function(container, application) {
   if (ENV.environment !== 'production') {
    window.ga = function() {};
    return;
   }
   
  application.deferReadiness();
  var GA_TRACK_ID = ENV['ga-track-id'];
  var addAnalytics = function() {
    /* jshint ignore:start */
    (function(i,s,o,g,r,a){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
    (i[r].q=i[r].q||[]).push(arguments);},i[r].l=1*new Date();a=s.createElement(o),
    a.async=1;a.src=g;s.body.appendChild(a);
    })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');
    /* jshint ignore:end */
    ga('create', GA_TRACK_ID, 'auto');

    application.advanceReadiness();
  };

  if (window.addEventListener) {
    window.addEventListener('load', addAnalytics, false);
  } else if (window.attachEvent) {
    window.attachEvent('onload', addAnalytics);
  } else {
    window.onload = addAnalytics;
  }
};

export default {
  name: 'analytics',
  initialize: initialize
};
```

Uygulama `production` ortamında çalışmadığı sürece Google analitik yüklenmeyecek fakat uygulama içinde ki kodu değiştirmemek ve hata almamamız için `ga` fonksiyonunu `window` objesine boş fonksiyon olarak atıyoruz.

Google analitik asenkron olarak yükleneceğinden `application.deferReadiness()` fonksiyonu çağırıyoruz böylece `application.advanceReadiness()` fonksiyonunu çağırana kadar uygulamamız UI render etme işlemine başlamayacak. Bunu yapmamızın sebebi Google analitik kodunun asenkron çalışması.

Uygulama ilk çalıştığında `ApplicationRoute` aktif hale gelip application templateini render edecek. Böylece analitik kodunu `ApplicationRoute` içine koyarsak kullanıcılarımızın hangi sayfalarına gittiğini kolaylıkla analiz edebiliriz

```javascript
import Em from 'ember';

export default Em.Route.extend({
  actions: {
    didTransition: function() {
      Em.run.once(this, function() {
        ga('send', 'pageview', this.router.get('url'));
      });
    }
  }
});
```

Yukarıdaki kod sayesinde uygulamada hangi her bir `route` ziyaret edildiğinde url Google analitiğe gönderilmiş olacak.

`environment` değişkeninde Google analitikten aldığımız id yi nasıl sakladığımızı merak edenler için aşağıda paylaşıyorum (Diğer uygulama ortam değişkenlerini bir karışıklık olmasın diye siliyorum)

```javascript
module.exports = function(environment) {
  var ENV = {
    modulePrefix: 'app',
    environment: environment,
    baseURL: '/',
    locationType: 'auto',
    EmberENV: {
      FEATURES: {
        // Here you can enable experimental features on an ember canary build
        // e.g. 'with-controller': true
      }
    },

    APP: {
      // Here you can pass flags/options to your application instance
      // when it is created
    }
  };
  if (environment === 'development') {
    ENV['ga-track-id'] = '';
  }
  if (environment === 'test') {
    ENV['ga-track-id'] = '';
  }

  if (environment === 'production') {
    ENV['ga-track-id'] = 'XXXXXXXXXXXX';
  }

  return ENV;
};
```
