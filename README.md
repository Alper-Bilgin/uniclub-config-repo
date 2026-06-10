# UniClub Mikrohizmet Konfigürasyon Deposu (uniclub-config-repo)

Bu depo, **UniClub** platformunun mikrohizmet mimarisine ait tüm yapılandırma dosyalarını (`.yml`) barındıran merkezi bir **Spring Cloud Config Server** deposudur. Sistemdeki tüm mikrohizmetler, çalışma zamanı (runtime) parametrelerini, veritabanı şemalarını, kuyruk yapılandırmalarını ve güvenlik anahtarlarını bu depodan çekerek dinamik olarak yapılandırılır.

---

##  Sistem Mimarisi & Rota Haritası

Aşağıdaki diyagramda, istemcilerin **API Gateway** üzerinden servislere nasıl eriştiği, servislerin **Eureka Discovery** üzerinde nasıl konumlandığı ve ortak altyapı bileşenleri ile olan ilişkileri gösterilmiştir:

```mermaid
graph TD
    Client[İstemci / Web UI / Mobil Uygulama] -->|Port 8080| Gateway[API Gateway]
    
    Gateway -->|/api/auth/**<br>/api/requests/**<br>/api/admin/**| AuthService[Auth Service :9001]
    Gateway -->|/api/profiles/**| ProfileService[UserProfile Service :9002]
    Gateway -->|/api/clubs/**| ClubService[Club Service :9003]
    Gateway -->|/api/events/**| EventService[Event Service :9004]
    Gateway -->|/api/registrations/**| RegService[Registration Service :9005]
    Gateway -->|/api/notifications/**| NotificationService[Notification Service :9006]
    Gateway -->|/api/posts/**| PostService[Post Service :9007]
    Gateway -->|/api/interactions/**| InteractionService[Interaction Service :9008]
    Gateway -->|/api/follows/**| FollowService[Follow Service :9009]
    Gateway -->|/api/feed/**| FeedService[Feed Service :9010]
    Gateway -->|/api/gamification/**| GamificationService[Gamification Service :9011]
    Gateway -->|/api/chat/** (HTTP)<br>/ws/** (WebSocket)| ChatService[Chat Service :9012]
    
    subgraph Kayıt & Keşif
        Eureka[Eureka Discovery Server :8761]
    end
    
    subgraph Ortak Altyapı Bileşenleri
        DB[(PostgreSQL :5432<br>uniclub_db)]
        Redis[(Redis :6379)]
        RabbitMQ[RabbitMQ :5672]
        MinIO[(MinIO Object Storage :9000)]
        Mailpit[Mailpit SMTP :1025]
    end
    
    AuthService -.-> Eureka
    ProfileService -.-> Eureka
    ClubService -.-> Eureka
    EventService -.-> Eureka
    RegService -.-> Eureka
    NotificationService -.-> Eureka
    PostService -.-> Eureka
    InteractionService -.-> Eureka
    FollowService -.-> Eureka
    FeedService -.-> Eureka
    GamificationService -.-> Eureka
    ChatService -.-> Eureka
```

---

##  Servis Port ve Şema Dağılımı

Tüm servisler tek bir PostgreSQL veritabanını (`uniclub_db`) kullanmakta, ancak veri yalıtımı (isolation) ve bağımsızlık sağlamak amacıyla **ayrı şemalara (schemas)** yazmaktadır.

| Servis Adı | Port | Veritabanı Şeması | Altyapı Bağımlılıkları | Açıklama |
| :--- | :---: | :---: | :--- | :--- |
| **API Gateway** | `8080` | *Yok* | Eureka | İstemci isteklerini karşılayan ve ilgili servislere yönlendiren geçit. |
| **Auth Service** | `9001` | `auth_schema` | PostgreSQL, RabbitMQ | Kimlik doğrulama, JWT üretimi ve kullanıcı kayıt işlemleri. |
| **User Profile Service**| `9002` | `profile_schema`| PostgreSQL, RabbitMQ, MinIO | Kullanıcı profilleri, kişisel bilgiler ve profil resimlerinin yönetimi. |
| **Club Service** | `9003` | `club_schema` | PostgreSQL, RabbitMQ, MinIO | Öğrenci kulüpleri, üyelikler ve kulüp logolarının yönetimi. |
| **Event Service** | `9004` | `event_schema` | PostgreSQL, Redis, MinIO | Kulüp etkinlikleri ve etkinlik görsellerinin yönetimi. |
| **Registration Service**| `9005` | `registration_schema`| PostgreSQL, Redis, RabbitMQ | Etkinlik biletleme ve kayıt işlemleri. |
| **Notification Service**| `9006` | `notification_schema`| PostgreSQL, RabbitMQ, Mailpit | E-posta şablonları, bildirim geçmişi ve sistem içi bilgilendirmeler. |
| **Post Service** | `9007` | `post_schema` | PostgreSQL, RabbitMQ, MinIO | Gönderi paylaşımı, medya yükleme işlemleri. |
| **Interaction Service** | `9008` | `interaction_schema`| PostgreSQL | Gönderi beğenileri, yorumlar ve etkileşimler. |
| **Follow Service** | `9009` | `follow_schema` | PostgreSQL, Redis | Kullanıcı takip ağları ve takipçi ilişkileri. |
| **Feed Service** | `9010` | *Yok* | Redis, RabbitMQ | Kullanıcıların ana sayfa akışlarının (feed) Redis üzerinde hızlıca oluşturulması. |
| **Gamification Service**| `9011` | `gamification_schema`| PostgreSQL, RabbitMQ | Rozet, puan ve oyunlaştırma kurgularının takibi. |
| **Chat Service** | `9012` | `chat_schema` | PostgreSQL, Redis, RabbitMQ | Canlı mesajlaşma (WebSocket) ve geçmiş sohbet yönetimi. |

---

##  Ortak Yapılandırma (`application.yml`)

Tüm mikrohizmetler tarafından miras alınan (inherit edilen) temel ayarlar `application.yml` dosyası içerisinde tanımlanmıştır:

*   **Eureka Discovery:** Servislerin Eureka Sunucusuna kaydolması (`defaultZone: http://localhost:8761/eureka/`) ve IP adresi ile haberleşmesi aktiftir.
*   **PostgreSQL:** Lokal ortam için varsayılan `username: postgres` ve `password: 1` olarak ayarlanmıştır. Hibernate `ddl-auto: update` modunda çalışmaktadır.
*   **RabbitMQ:** Olay tabanlı (event-driven) haberleşme için `localhost:5672` (guest/guest) adresi üzerinden bağlanır.
*   **Redis:** Akış (feed) yönetimi, oturum/çevrimiçi durum takibi ve cacheleme için `localhost:6379` adresi aktiftir.

---

##  Güvenlik & JWT Ayarları

Kimlik doğrulama mekanizması olarak **JWT (JSON Web Token)** kullanılmıştır. Aşağıdaki ayarlar tüm güvenlik doğrulayan servislerde ortaktır:

*   **JWT Secret:** `bmljZS1zZWNyZXQtZm9yLXVuaWNsdWItY29ubmVjdC1wcm9qZWN0LXdpdGgtZW5vdWdoLWJ5dGVzLWZvci1oczUxMg==` (Base64 kodlu HS512 anahtarı)
*   **Access Token Ömrü:** 1 Saat (`3600000 ms`)
*   **Refresh Token Ömrü:** 7 Gün (`604800000 ms`)

---

##  MinIO Nesne Depolama Yapılandırması

Medya dosyalarının (görsel, logo vb.) saklanması için lokalde veya Docker üzerinde `http://localhost:9000` adresinde çalışan **MinIO** kullanılmaktadır (Giriş bilgileri: `minioadmin` / `minioadmin`). 

Her servis kendine ayrılmış özel bir kova (bucket) kullanır:
*   `uniclub-profiles`: Profil fotoğrafları (**User Profile Service**)
*   `uniclub-logos`: Kulüp logoları (**Club Service**)
*   `uniclub-events`: Etkinlik afişleri (**Event Service**)
*   `uniclubposts`: Gönderi medyaları (**Post Service**)

---

##  RabbitMQ Mesajlaşma Topolojisi

Sistemdeki servisler arası asenkron iletişim RabbitMQ üzerinden Event-Driven (Olay Güdümlü) yaklaşımla sağlanmaktadır. 

### Yayınlanan Olaylar (Publishers)
1.  **Auth Service:**
    *   **Exchange:** `user_exchange`
    *   **Routing Key:** `user.created.key` (Yeni kullanıcı kaydı olayını tetikler)
    *   **Exchange (Oyunlaştırma):** `gamification.exchange`
    *   **Routing Key:** `gamification.event.user.login` (Kullanıcı giriş yaptığında tetiklenir)
2.  **Club Service:**
    *   **Exchange:** `club_exchange`
    *   **Routing Key:** `user.joined.club.key` (Kullanıcı kulübe katıldığında tetiklenir)
3.  **Post Service:**
    *   **Exchange:** `post.exchange`
    *   **Routing Key:** `post.created` ve `post.deleted` (Gönderi ekleme ve silme)
    *   **Exchange (Oyunlaştırma):** `gamification.exchange`
    *   **Routing Key:** `gamification.event.post.created` (Gönderi paylaşıldığında oyunlaştırma puanı için tetiklenir)
4.  **Registration Service:**
    *   **Exchange:** `notification_exchange`
    *   **Routing Key:** `ticket.created.key` (Kullanıcı bilet aldığında e-posta için tetiklenir)

### Dinlenen Kuyruklar & Alıcılar (Subscribers)
1.  **User Profile Service:**
    *   **Kuyruk:** `profile_user_created_queue` -> `user_exchange` üzerinden gelen `user.created.key` olayını dinleyerek otomatik profil şablonu oluşturur.
2.  **Feed Service:**
    *   **Kuyruk:** `feed_post_event_queue` -> `post.exchange` üzerinden gelen `post.created` ve `post.deleted` olaylarını dinleyerek Redis akış listesini günceller.
3.  **Gamification Service:**
    *   **Kuyruk:** `gamification.events.queue` -> `gamification.exchange` altındaki `gamification.event.#` şablonuna uyan tüm oyunlaştırma olaylarını dinler.
4.  **Notification Service:**
    *   Sistemde e-posta veya push bildirim göndermek için aşağıdaki kuyrukları dinler:
        *   `notification_welcome_email_queue` (`user_exchange` -> `user.created.key`)
        *   `notification_ticket_email_queue` (`notification_exchange` -> `ticket.created.key`)
        *   `notification_follow_email_queue` (`follow.exchange` -> `follow.created`)
        *   `notification_chat_message_queue` (`chat_exchange` -> `chat.message.unread`)
        *   `notification_password_reset_queue` (`user_exchange` -> `user.reset.key`)

---

##  E-posta Yapılandırması (Mailpit)

**Notification Service**, geliştirme ortamında e-postaların gerçek kişilere gitmesini önlemek ve güvenle test edebilmek için bir mock mail aracı olan **Mailpit**'e yönlendirilmiştir:
*   **SMTP Sunucusu:** `localhost`
*   **SMTP Portu:** `1025`
*   **Kimlik Doğrulama:** Pasif (`auth: false`, `starttls: false`)

---

##  Spring Cloud Config Sunucusunda Nasıl Kullanılır?

Bu depodaki yapılandırmaları kullanabilmek için Spring Cloud Config Server projenizin `application.yml` veya `bootstrap.yml` dosyasına bu deponun yolunu göstermeniz gerekir:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/Alper-Bilgin/uniclub-config-repo.git # ya da lokal path
          clone-on-start: true
```

Lokal geliştirme ortamında doğrudan bilgisayarınızdaki dizini hedeflemek isterseniz:

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: file://${user.home}/Desktop/Bitirme Projesi/uniclub-config-repo
```
##  Geliştirici

**Alper Bilgin**
-  [Bağlantılarım (Linktree)](https://linktr.ee/Alper_Bilgin)