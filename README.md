# Kubernetes Manifestleri, Kustomize ve GitHub Actions Pipeline
Bu repo, 'simple-todo-app' adlı örnek bir uygulamanın Kubernetes manifestlerini içermektedir. Proje, "her şeyi kod olarak yapma", "doğru iş için doğru aracı kullanma" ve "modüler, genişleyebilir bir yapı oluşturma" felsefeleriyle tasarlanmıştır.

Bu repo, uygulamanın farklı ortamlar (geliştirme ve üretim) için özelleştirilmiş Kubernetes dağıtımlarını yönetmek için Kustomize kullanır. Ayrıca, uygulamanın sürekli entegrasyon ve sürekli dağıtım (CI/CD) süreçlerini otomatize eden bir GitHub Actions pipeline'ını da içerir.

## **İçerik**
1. Proje Yapısı
2. Kustomize Kullanımı
    - Temel (Base) Manifestler
    - Ortam Katmanları (Overlays)
3. GitHub Actions CI/CD Pipeline
    - Pipeline Tetikleyicileri
    - Ortam Değişkenleri
    - Pipeline Adımları (Jobs)
        - build-and-test Job*
        - deploy-to-dev Job
        - dynamic-analysis-DAST Job
        - manual-approval-for-prod Job
        - deploy-to-prod Job
4. Kurulum ve Kullanım
    - Önkoşullar
    - Yerel Ortamda Kustomize Uygulama
    - GitHub Actions Yapılandırması
5. Önemli Notlar ve Varsayımlar

## 1. Proje Yapısı
- Bu repo, Kustomize'ın standart dizin yapısını takip eder.
- base dizini uygulamanın temel Kubernetes manifestlerini içerirken, overlays dizini farklı ortamlar (dev, prod) için bu manifestlere yapılan yamaları (patch) barındırır.

/manifest-mehmetkanus:
.
├── base
│   ├── deployment.yaml
│   ├── external-secret.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── deployment-patch.yaml
    │   ├── external-secret-patch.yaml
    │   ├── hpa-patch.yaml
    │   ├── ingress-patch.yaml
    │   └── kustomization.yaml
    └── prod
        ├── deployment-patch.yaml
        ├── external-secret-patch.yaml
        ├── hpa-patch.yaml
        ├── ingress-patch.yaml
        └── kustomization.yaml

## 2. Kustomize Kullanımı
Kustomize, Kubernetes uygulamalarını bir şablonlama motoru kullanmadan özelleştirmenize olanak tanır. Mevcut Kubernetes manifestlerini "yamayarak" farklı ortamlar için farklı konfigürasyonlar oluşturmayı kolaylaştırır.

**Temel (Base) Manifestler**
- base dizini, simple-todo-app uygulamasının tüm ortamlar için ortak olan temel Kubernetes kaynaklarını (Deployment, Service, Ingress, HPA, External Secret) tanımlar.

- deployment.yaml: Uygulamanın pod'larını ve replikalarını tanımlar.

- external-secret.yaml: Vault gibi bir secret yönetim sisteminden Kubernetes secret'larını çekmek için ExternalSecrets operator ile kullanılır.

- hpa.yaml: Uygulamanın Horizontal Pod Autoscaler ayarlarını içerir.

- ingress.yaml: Uygulamaya dışarıdan erişim sağlamak için Ingress kuralını tanımlar.

- service.yaml: Uygulama pod'larına ağ erişimi sağlayan Kubernetes Service'i tanımlar.

- kustomization.yaml: base dizinindeki kaynakları listeler.

**Ortam Katmanları (Overlays)**
- overlays dizini, base manifestler üzerine uygulanan ortama özgü yamaları içerir. Her alt dizin (dev, prod), belirli bir ortam için özelleştirmeleri barındırır.

- overlays/dev/: Geliştirme ortamı için yamalar ve kustomization.yaml.

- overlays/prod/: Üretim ortamı için yamalar ve kustomization.yaml.

- Her kustomization.yaml dosyası, base dizinindeki kaynakları içe aktarır ve ortama özgü yamaları (örneğin, replika sayısı, etiketler, host adları, resource limitleri, image tag'leri) uygular. Özellikle, uygulamanın imaj tag'i ve ortam spesifik URL'leri gibi değerler bu yama dosyaları aracılığıyla güncellenir.

- Örnek overlays/dev/kustomization.yaml içindeki image tag güncellemesi:

````yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patchesStrategicMerge:
  - deployment-patch.yaml
  - ingress-patch.yaml
  - hpa-patch.yaml
  - external-secret-patch.yaml
images:
  - name: harbor.prod.goxdev.lol/app-mehmetkanus/todoapp # Base'de tanımlı imaj adı
    newTag: <GITHUB_SHA> # GitHub Actions tarafından güncellenecek
````

## 3. GitHub Actions CI/CD Pipeline

Bu repo, simple-todo-app için kapsamlı bir CI/CD pipeline'ını otomatikleştirmek üzere tasarlanmış bir GitHub Actions iş akışını içerir. Pipeline, kodun push edilmesinden üretim ortamına dağıtımına kadar tüm süreci yönetir.

Pipeline dosyası: .github/workflows/main.yaml

**Pipeline Tetikleyicileri**

Pipeline, aşağıdaki durumlarda tetiklenir:
push olayları: main veya feature branch'lerine kod push edildiğinde.
pull_request olayları: main veya feature branch'lerine bir Pull Request açıldığında veya güncellendiğinde.
workflow_dispatch: GitHub Actions arayüzünden manuel olarak tetiklenebilir. Bu, belirli bir ortamı hedeflemek veya sadece belirli adımları çalıştırmak için kullanılır.
run-options: Pipeline'ın tüm adımları çalıştırmasını (run-all) veya sadece build ve test adımını (only-build-job) çalıştırmasını seçmenizi sağlar.

**Ortam Değişkenleri**

- Pipeline genelinde kullanılan önemli ortam değişkenleri:
- APP_NAME: Uygulama adı (simple-todo-app).
- HARBOR_URL: Harbor imaj kayıt defterinin URL'si (harbor.prod.goxdev.lol).
- APP_IMAGE: Uygulama imajının tam yolu (harbor.prod.goxdev.lol/app-mehmetkanus/todoapp).
- GITOPS_REPO: Bu manifest reposunun GitHub yolu (${{ github.repository_owner }}/manifest-mehmetkanus).
- DOCKERFILE_PATH: Dockerfile'ın yolu (./SimpleTodoApp/Dockerfile).
- APP_SOURCE_DIR: Uygulama kaynak kodunun dizini (./SimpleTodoApp).
- DEV_APP_URL: Geliştirme ortamı uygulama URL'si (app.dev.goxdev.lol).
- PROD_APP_URL: Üretim ortamı uygulama URL'si (app.prod.goxdev.lol).

**Pipeline Adımları (Jobs)**

***build-and-test Job***
- Bu job, uygulamanın kaynak kodunu derler, test eder ve Docker imajını oluşturup Harbor'a push'lar. Ayrıca statik kod analizi (SAST) ve imaj güvenlik taraması yapar.
- Checkout Source Code: repoyu klonlar.
- Trivy Filesystem Scan (SAST): Kaynak kodunda güvenlik açıklarını taramak için Trivy kullanılır. Kritik ve yüksek önem dereceli açıklarda pipeline durur (exit-code 1).
- Login to Harbor: Docker imaj kayıt defterine (Harbor) giriş yapar. Bunun için GitHub Actions secrets (HARBOR_USERNAME, HARBOR_PASSWORD) kullanılır.
- Extract metadata (tags, labels) for Docker: Docker imajı için github.sha (commit hash) ve latest etiketlerini belirler.
- Build and Push Docker Image to Harbor: Uygulama Dockerfile'ı kullanılarak imaj oluşturulur ve belirlenen etiketlerle Harbor'a push'lanır.
- Trivy Image Scan (SAST - Image): Oluşturulan Docker imajında güvenlik açıklarını taramak için Trivy kullanılır. Kritik ve yüksek önem dereceli açıklarda uyarı verir (exit-code 0).

***deploy-to-dev Job***
- Bu job, build-and-test job'ı başarıyla tamamlandıktan sonra uygulamanın en yeni imajını geliştirme ortamına dağıtır.
- Bu işlem, GitOps prensipleriyle çalışır ve bu repodaki geliştirme ortamı kustomization.yaml dosyasındaki imaj etiketini güncelleyerek ArgoCD gibi bir CD aracının değişiklikleri algılayıp dağıtmasını sağlar.
- Update GitOps deployment repo: Bu repo (manifest-mehmetkanus) klonlanır.
- Kustomization.yaml Güncelleme: overlays/dev/kustomization.yaml dosyasındaki newTag alanı, en son github.sha (commit hash) ile güncellenir.
- Commit and Push: Güncellenen kustomization.yaml dosyası commit edilir ve main branch'ine push edilir. Bu push işlemi, ArgoCD'yi tetikleyerek geliştirme ortamına otomatik dağıtımı başlatır.

***dynamic-analysis-DAST Job***
- Geliştirme ortamına dağıtım yapıldıktan sonra, uygulamanın canlı ortamdaki güvenlik açıklarını tespit etmek için dinamik güvenlik analizi (DAST) yapılır.
- Uygulamanın hazır olmasını bekle: Dağıtılan uygulamanın erişilebilir olmasını sağlamak için kısa bir bekleme süresi vardır.
- ZAP Baseline Scan (DAST): OWASP ZAP kullanılarak geliştirme ortamındaki uygulamanın URL'sinde (dev.goxdev.lol) bir temel güvenlik taraması gerçekleştirilir. GITOPS_PAT secret'ı, GitHub Issues oluşturma izni için kullanılır.

***manual-approval-for-prod Job***
- Üretim ortamına dağıtımdan önce manuel bir onay adımı eklenmiştir. Bu, üretim ortamına yapılan dağıtımların kontrol altında tutulmasını sağlar.
- Manual approval to deploy to production: trstringer/manual-approval eylemi kullanılır. Belirlenen onaylayıcılar (örneğin, mehmetkanus17) tarafından bir GitHub Issue üzerinden onay beklenecektir. Onay verilmezse pipeline durur.
- Onay isteyen bir GitHub Issue oluşturulur.
- DAST raporlarına (geçerli GitHub Actions run linki) referans verilir.

***deploy-to-prod Job***
- Manuel onay verildikten sonra, uygulamanın en yeni imajı üretim ortamına dağıtılır. Bu adım da GitOps prensipleriyle çalışır.
- Update GitOps deployment repo: Bu repo (manifest-mehmetkanus) klonlanır.
- Kustomization.yaml Güncelleme: overlays/prod/kustomization.yaml dosyasındaki newTag alanı, onaylanan github.sha (commit hash) ile güncellenir.
- Commit and Push: Güncellenen kustomization.yaml dosyası commit edilir ve main branch'ine push edilir. Bu push işlemi, ArgoCD'yi tetikleyerek üretim ortamına otomatik dağıtımı başlatır.

## 4. Kurulum ve Kullanım
Bu repoyu kullanmak ve CI/CD pipeline'ını çalıştırmak için aşağıdaki adımları izleyin.

**Önkoşullar**
- Bir Kubernetes kümesi (dev ve prod namespace'leri ile).
- ArgoCD kurulu ve GitOps prensiplerine göre yapılandırılmış.
- Harbor veya benzeri bir Docker imaj kayıt defteri kurulu.
- Hashicorp Vault kurulu ve Kubernetes ile entegre edilmiş.
- External Secrets Operator kurulu ve Vault ile iletişim kuracak şekilde yapılandırılmış.
- GitHub Personal Access Token (PAT): Bu repo için secrets.GITOPS_PAT olarak kullanılacak ve repo içeriğini okuma/yazma, GitHub Issues oluşturma/güncelleme izinlerine sahip olmalıdır.
- Harbor Kullanıcı Adı ve Parolası: GitHub Actions secrets (HARBOR_USERNAME, HARBOR_PASSWORD) olarak yapılandırılmalıdır.
- simple-todo-app uygulamasının kaynak kodu ve Dockerfile'ı (APP_SOURCE_DIR ve DOCKERFILE_PATH tarafından belirtilen yolda) mevcut olmalıdır.

**Yerel Ortamda Kustomize Uygulama**
- Uygulama manifestlerini yerel olarak test etmek veya incelemek için Kustomize'ı kullanabilirsiniz:
- Geliştirme Ortamı Manifestlerini Oluşturma:

````bash
kubectl kustomize overlays/dev
````
- Bu komut, base manifestlerini dev ortamı yamalarıyla birleştirerek nihai Kubernetes manifestlerini stdout'a yazdırır.
- Üretim Ortamı Manifestlerini Oluşturma:

````bash
kubectl kustomize overlays/prod
````
- Bu komut da aynı şekilde base manifestlerini prod ortamı yamalarıyla birleştirir.

- Bu manifestleri Kubernetes kümenize uygulamak için çıktıyı bir dosyaya yönlendirebilir ve kubectl apply -f ile uygulayabilirsiniz:

````
kubectl kustomize overlays/dev > dev-manifests.yaml
kubectl apply -f dev-manifests.yaml
````

**GitHub Actions Yapılandırması**
- GitHub Actions pipeline'ının sorunsuz çalışması için aşağıdaki GitHub Secrets'ı reponuzda tanımlamanız gerekmektedir:

- HARBOR_USERNAME: Harbor kayıt defterinize erişim için kullanıcı adı.
- HARBOR_PASSWORD: Harbor kayıt defterinize erişim için parola.
- GITOPS_PAT: GitHub Personal Access Token. Bu token, pipeline'ın bu repoyu klonlaması, kustomization.yaml dosyalarını güncellemesi ve GitHub Issues (manuel onay için) oluşturması/güncellemesi için kullanılır.

- Repo: contents: write
- Issues: write

-**Pipeline'ı Tetikleme**
- **Kod Değişikliği ile Tetikleme:** SimpleTodoApp reposunda (veya uygulamanızın kaynak kodu reposunda) main veya feature branch'ine bir kod değişikliği push yaptığınızda, ilgili CI/CD pipeline otomatik olarak tetiklenecektir.

- **Manuel Tetikleme:** GitHub reposunun "Actions" sekmesine giderek "CI/CD Pipeline for Application" iş akışını seçebilir ve "Run workflow" düğmesine tıklayarak manuel olarak tetikleyebilirsiniz. run-options seçeneğini kullanarak only-build-job veya run-all seçeneklerinden birini belirleyebilirsiniz.

## 5. Önemli Notlar ve Varsayımlar

**GitOps:**
- Dağıtım süreçleri (dev ve prod ortamlarına), ArgoCD gibi bir GitOps aracı tarafından bu repodaki kustomization.yaml dosyalarındaki değişikliklerin izlenmesiyle otomatik olarak gerçekleşir. GitHub Actions sadece kustomization.yaml'daki imaj etiketini güncelleyerek bir commit push eder.

**Secret Yönetimi:**
- Hassas bilgiler (veritabanı kimlik bilgileri vb.) Vault'ta saklanır ve Kubernetes kümesine External Secrets Operator aracılığıyla güvenli bir şekilde aktarılır. Git repolarında hiçbir secret plaintext olarak bulunmaz.

**Güvenlik Tarama Araçları:**

- **SAST (Static Application Security Testing):** Kaynak kodu ve Docker imajları, geliştirme yaşam döngüsünün erken aşamalarında güvenlik açıklarını bulmak için Trivy ile taranır. SonarQube entegrasyonu da yorum satırı olarak mevcuttur.

- **DAST (Dynamic Application Security Testing):** Dağıtılmış uygulama üzerinde dinamik testler yapmak için OWASP ZAP kullanılır.

**Manuel Onay:**
- Üretim dağıtımları için ek bir güvenlik katmanı olarak manuel onay adımı zorunludur.

**Ortam Ayrımı:**
- Geliştirme ve üretim ortamları, Kubernetes namespace'leri ve Kustomize overlays kullanılarak mantıksal olarak ayrılmıştır.

**Uygulama:**
- simple-todo-app, ilişkisel bir veritabanı kullanan, durumu koruyan (stateful) ve bir kullanıcı arayüzü olan basit bir uygulamadır. Bu repo sadece uygulamanın Kubernetes manifestlerini içerir, uygulamanın kendi kaynak kodunu içermez.

**Container Registry:**
- Harbor, Docker imajlarını repolamak ve yönetmek için kullanılır.