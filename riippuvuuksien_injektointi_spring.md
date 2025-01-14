---
layout: page
title: Riippuvuuksien injektointi Spring-sovelluskehyksessä
inheader: no
permalink: /riippuvuuksien_injektointi_spring/
---

## Dependency injection Spring-sovelluskehyksessä

Jatketaan viime viikolla käsittelemämme [laskimen](/riippuvuuksien_injektointi/) tarkastelua. Kertaa tarvittaessa yllä olevan linkin takana oleva teksti.  

Alla oleva koodi löytyy gradle-muotoisina projekteina kurssin [tehtävärepositoriosta](https://github.com/ohjelmistotuotanto-hy/syksy2019) hakemistosta koodi/viikko2/RiippuvuuksienInjektointiSpring

Päädyimme siis tilanteeseen, missä Laskin-luokasta on erotettu konkreettinen riippuvuus syötteen lukemiseen ja tulostamiseen. Laskin tuntee ainoastaan _rajapinnan_ <code>IO</code> jonka kautta se hoitaa syötteen käsittelyn ja tulostamisen. 

Ennen käynnistämistä refakotoroitu laskin pitää _konfiguroida_ injektoimalla sille sopivat riippuvuudet:

``` java
// konfigurointivaihe
Laskin laskin = new Laskin( new KonsoliIO() );

// ja laskin valmiina käyttöön
laskin.suorita();
```

Esimerkkimme tapauksessa konfigurointi on helppoa. Isommissa ohjelmissa konfiguroitavalla oliolla voi olla suuri määrä riippuvuuksia ja konfigurointi voi olla monimutkaista.

Kurssilta [Web-palvelinohjelmointi](https://courses.helsinki.fi/fi/tkt21007) osalle tuttu [Spring-sovelluskehys](http://www.springsource.org/) tarjoaa mekanismin, jonka avulla riippuvuuksien injektointi voidaan siirtää Springin vastuulle. 

> Spring on laaja ja monikäyttöinen sovelluskehys, jota käytetään yleisesti mm. Javalla tapahtuvassa web-sovelluskehityksessä. Tutustumme kurssilla oikeastaan ainoastaan Springin riippuvuuksien injektointiin. 

Spring saadaan käyttöön lisäämällä Spring riippuvuudeksi gradle-projektin määrittelemään build.gradle-tiedostoon, katso tarkemmin [täältä](https://github.com/ohjelmistotuotanto-hy/syksy2019/tree/master/koodi/viikko2/RiippuvuuksienInjektointiSpring)

Springissä ideana on siirtää osa sovelluksen olioista ns. [Inversion of Control container](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/core.html#beans-basics):in eli eräänlaisen olioisäiliönä toimivan sovelluskontekstin hallinnoitavaksi. 

Springissä on kaksi tapoja määritellä sovelluskontekstin oliot. Käytämme nykyään suositumpaa  [annotaatioihin](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/core.html#beans-annotation-config) perustuvaa määrittelytapaa. 

Sovelluskontekstin huolehdittavaksi annettavien olioiden luokat merkitään annotaatiolla  <code>@Component</code>. Luokka _KonsoliIO_ annotoidaan seuraavasti:

``` java
@Component
public class KonsoliIO implements IO {
  // ...
}
```

Myös luokka _Laskin_ merkataan annotaatiolla _Component_. Tämän lisäksi Springille kerrotaan annotaatiolla <code>@Autowired</code>, että sen täytyy injektoida laskimelle rajapinnan _IO_ toteuttava olio konstruktoriparametrina:

``` java
@Component
public class Laskin {
    private IO io;

    @Autowired
    public Laskin(IO io) {
        this.io = io;
    }

  //...

}
```

Itseasiassa samaan lopputulokseen päästäisiin vieläkin helpommin merkkaamalla laskimen oliomuuttuja _io_ annotaatiolla <code>@Autowired</code>, tällöin konstruktoria ei tarvita:

``` java
@Component
public class Laskin {
    @Autowired
    private IO io;
    
    // ...      
   
}
``` 

Sovellus on vielä konfiguroitava, jotta Spring tietää, että käytössä on annotaatiopohjainen määrittely. Konfigurointi voidaan tehdä joko xml-tiedostona tai [Java-luokkana](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/core.html#beans-java). 

Käytetään luokkapohjaista tapaa. Konfiguroinnin hoitava luokka on yksinkertainen:

``` java
@Configuration
@ComponentScan(basePackages = "ohtu.laskin")
public class AppConfig  {}
```

Käytännössä luokka siis määrittelee, että käytetään annotaatiopohjaista konfiguraatiota, ja etsitään annotoituja luokkia pakkauksen _ohtu.laskin_ alta.

Pääohjelma on nyt seuraava:

``` java
public class Main {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

        Laskin laskin = ctx.getBean(Laskin.class)
        laskin.suorita();
    }
}
```

Ensimmäinen rivi luo sovelluskontekstin. Tämän jälkeen kontekstilta pyydetään _Laskin_-olio ja kutsutaan laskimen käynnistävää metodia. 

Spring siis luo konfiguraatioiden ansiosta automaattisesti laskimen ja injektoi sille _KonsoliIO_-olion. 

Oletusarvoisesti Spring luo ainoastaan _yhden_ olion kustakin luokasta, eli jos metodikutsu _ctx.getBean(Laskin.class)_ suoritetaan toistuvasti, se palauttaa aina saman olion. 

Jos tämä ei ole haluttu toimintatapa, voidaan Spring [konfiguroida](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/core.html#beans-scanning-scope-resolver) palauttamaan jokaisella kutsulla uusi olio.

Lisää tieto Springin kontainereiden toiminnasta löytyy [dokumentaatiosta](https://docs.spring.io/spring/docs/5.2.0.RELEASE/spring-framework-reference/core.html#beans).