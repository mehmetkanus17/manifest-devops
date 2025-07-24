# manifest-mehmetkanus: Kubernetes Manifestleri

Bu repo, DevOps proje kapsamında, `SimpleTodoApp` uygulamasının Kubernetes manifestlerini içermektedir. Repo, Kustomize kullanarak farklı ortamlar (`dev` ve `prod`) için uygulama dağıtımlarını yönetir. Bu yapı, Infrastructure as Code (IaC) ve GitOps prensiplerine uygun olarak, uygulama dağıtımlarının versiyonlanabilir, tekrarlanabilir ve otomatik olmasını sağlar.

## İçindekiler

1.  Genel Bakış
2.  Dizin Yapısı
3.  Kustomize ile Ortam Yönetimi
    *   Base Manifestler
    *   Overlay Manifestler
4.  Altyapı (Infra) ile Entegrasyon
5.  CI/CD Süreci ile Entegrasyon
6.  Kustomize Gereksinimleri ile Uyum

## 1. Genel Bakış

`manifest-mehmetkanus` reposu, `SimpleTodoApp` uygulamasının Kubernetes dağıtım manifestlerini merkezi bir konumda tutar. Bu repo, Kustomize kullanılarak `dev` ve `prod` gibi farklı ortamlar için uygulama yapılandırmalarını yönetir. Bu yaklaşım, aynı temel manifestleri kullanarak ortama özgü farklılıkları (örneğin, replika sayısı, kaynak limitleri, External Secret yapılandırmaları) kolayca yönetmeyi sağlar.

Reponun temel amacı, Kubernetes dağıtımlarını "as code" prensibiyle yönetmek ve GitOps iş akışlarını desteklemektir. Bu sayede, uygulama dağıtımları Git reposundaki değişikliklerle otomatik olarak senkronize edilir, bu da dağıtım süreçlerini daha güvenilir ve şeffaf hale getirir.

Bu repo, aşağıdaki temel prensiplere dayanmaktadır:

*   **Tek Kaynak Gerçeği (Single Source of Truth)**: Tüm Kubernetes manifestleri bu repoda bulunur ve versiyon kontrolü altındadır.
*   **Ortam Ayrımı**: Kustomize overlayleri aracılığıyla `dev` ve `prod` ortamları birbirinden ayrılırken, temel manifestler yeniden kullanılır.
*   **Otomatik Dağıtım**: CI/CD pipeline (özellikle CD kısmı), bu repodaki değişiklikleri izleyerek uygulamaların Kubernetes kümelerine otomatik olarak dağıtılmasını tetikler.
*   **Güvenli Sır Yönetimi**: External Secrets entegrasyonu sayesinde hassas veriler (sırlar) doğrudan manifestlerde saklanmaz, bunun yerine HashiCorp Vault gibi harici bir sır yönetim sisteminden çekilir.

Bu README, reponun yapısını, Kustomize kullanımını ve önceki altyapı ve CI/CD süreçleriyle nasıl entegre olduğunu detaylandıracaktır.

## 2. Dizin Yapısı

`manifest-mehmetkanus` reposunun dizin yapısı, Kustomize standartlarına uygun olarak düzenlenmiştir:

```
.
├── README.md
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
```

*   **`base/`**: Uygulamanın temel Kubernetes manifestlerini içerir. Bu manifestler, herhangi bir ortama özgü yapılandırma içermez ve tüm ortamlar için ortak olan ayarları tanımlar.
    *   `deployment.yaml`: Uygulamanın Deployment tanımı.
    *   `external-secret.yaml`: Uygulamanın ihtiyaç duyduğu sırları Vault gibi harici bir kaynaktan çeken ExternalSecret tanımı.
    *   `hpa.yaml`: Uygulamanın Horizontal Pod Autoscaler (HPA) tanımı.
    *   `ingress.yaml`: Uygulamanın dışarıdan erişimini sağlayan Ingress tanımı.
    *   `kustomization.yaml`: `base` dizinindeki tüm manifestleri bir araya getiren Kustomize dosyası.
    *   `service.yaml`: Uygulamanın Service tanımı.

*   **`overlays/`**: Farklı ortamlar için ortama özgü yapılandırmaları içerir. Her bir alt dizin (`dev`, `prod`) belirli bir ortamı temsil eder.
    *   **`dev/`**: Geliştirme ortamına özgü yamaları ve `kustomization.yaml` dosyasını içerir.
        *   `deployment-patch.yaml`: `base` Deployment manifestine uygulanan yamalar (örneğin, replika sayısı, kaynak limitleri).
        *   `external-secret-patch.yaml`: `base` ExternalSecret manifestine uygulanan yamalar (örneğin, Vault yolu).
        *   `hpa-patch.yaml`: `base` HPA manifestine uygulanan yamalar.
        *   `ingress-patch.yaml`: `base` Ingress manifestine uygulanan yamalar.
        *   `kustomization.yaml`: `base` manifestlerini ve `dev` ortamına özgü yamaları bir araya getiren Kustomize dosyası. Bu dosya aynı zamanda imaj etiketini (`newTag`) de içerir ve CI/CD pipeline tarafından güncellenir.
    *   **`prod/`**: Üretim ortamına özgü yamaları ve `kustomization.yaml` dosyasını içerir. Yapısı `dev` dizini ile benzerdir, ancak üretim ortamına özgü değerler içerir.

Bu yapı, manifestlerin DRY (Don't Repeat Yourself) prensibine uygun olarak yönetilmesini ve ortamlar arası farklılıkların kolayca izlenmesini sağlar.

## 3. Kustomize ile Ortam Yönetimi

Bu repo, Kubernetes manifestlerini yönetmek için Kustomize kullanır. Kustomize, aynı temel manifestleri farklı ortamlara uyarlamak için yamalar (patches) ve transformasyonlar uygulayan bir araçtır. Bu sayede, her ortam için ayrı ayrı tam manifest dosyaları tutmak yerine, sadece ortamlar arası farklılıklar belirtilir.

### Base Manifestler

`base/` dizini, uygulamanın tüm ortamlar için ortak olan temel Kubernetes kaynaklarını içerir. Bu manifestler, uygulamanın çekirdek yapılandırmasını tanımlar ve herhangi bir ortama özgü bilgi içermez. Örneğin, bir Deployment manifesti temel konteyner imajını, portları ve genel etiketleri tanımlar.

### Overlay Manifestler

`overlays/` dizini altında, her bir ortam için ayrı bir alt dizin bulunur (`dev`, `prod`). Bu dizinler, `base` manifestlerine uygulanacak ortama özgü yamaları ve kendi `kustomization.yaml` dosyalarını içerir.

Her bir `overlays/<ortam>/kustomization.yaml` dosyası:

*   `base` dizinindeki manifestleri referans alır.
*   Ortama özgü yamaları (`-patch.yaml` dosyaları) `base` manifestlerine uygular. Bu yamalar, replika sayısı, kaynak limitleri, Ingress hostları veya External Secret referansları gibi değerleri değiştirebilir.
*   `newTag` alanı aracılığıyla uygulamanın Docker imaj etiketini günceller. Bu alan, CI/CD pipeline tarafından otomatik olarak güncellenir ve uygulamanın yeni versiyonlarının dağıtımını tetikler.

**Örnek Kustomize Yapısı (`overlays/dev/kustomization.yaml`):**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - path: deployment-patch.yaml
  - path: external-secret-patch.yaml
  - path: hpa-patch.yaml
  - path: ingress-patch.yaml

images:
  - name: app-mehmetkanus/todoapp # Base manifestteki imaj adı
    newTag: <CI/CD tarafından güncellenen imaj etiketi>

namespace: dev # Uygulamanın dağıtılacağı namespace
```

Bu yapı, manifestlerin yönetimini basitleştirir, hataları azaltır ve farklı ortamlar arasında tutarlılığı sağlar.

## 4. Altyapı (Infra) ile Entegrasyon

`manifest-mehmetkanus` reposundaki Kubernetes manifestleri, `infra-mehmetkanus` reposu tarafından kurulan altyapı ile sorunsuz bir şekilde entegre olacak şekilde tasarlanmıştır. Bu entegrasyon, uygulamanın Kubernetes kümesi üzerinde doğru ve güvenli bir şekilde çalışmasını sağlar.

Başlıca entegrasyon noktaları şunlardır:

*   **Kubernetes Kümesi**: Manifestler, `infra-mehmetkanus` tarafından Kubespray ile kurulan Kubernetes kümesi üzerinde dağıtılmak üzere tasarlanmıştır. `dev` ve `prod` ortamları, küme içindeki ayrı namespace'lerde (`dev` ve `prod`) bulunur.
*   **HAProxy ve Nginx Ingress Controller**: Uygulamanın `ingress.yaml` manifesti, `infra-mehmetkanus` tarafından kurulan Nginx Ingress Controller ve HAProxy yük dengeleyici ile uyumludur. Ingress kaynakları, uygulamanın dışarıdan erişilebilir olmasını sağlar.
*   **NFS Persistent Volume Claims (PVC)**: Uygulamanın veri kalıcılığı gereksinimleri için tanımlanan PVC'ler, `infra-mehmetkanus` tarafından kurulan NFS tabanlı repolama altyapısını kullanır. `nfs-subdir-external-provisioner` sayesinde dinamik olarak Persistent Volume (PV) oluşturulur.
*   **HashiCorp Vault ve External Secrets**: `external-secret.yaml` manifesti, `infra-mehmetkanus` tarafından kurulan HashiCorp Vault ve External Secrets Operator ile entegredir. Bu, uygulamanın hassas verilerini (örneğin veritabanı bağlantı dizeleri) doğrudan manifestlerde saklamak yerine, Vault'tan güvenli bir şekilde çekmesini sağlar. `external-secret-patch.yaml` dosyaları, ortama özel Vault yollarını belirtir.
*   **Service Accounts**: Uygulama Deployment'ı, `infra-mehmetkanus` tarafından oluşturulan `vault-serviceaccount` gibi özel Service Account'ları kullanabilir. Bu Service Account'lar, Vault ile kimlik doğrulaması yapmak ve gerekli izinleri almak için kullanılır.

Bu entegrasyon, uygulamanın temel altyapı servislerini (yük dengeleme, depolama, sır yönetimi) kullanarak güvenli ve verimli bir şekilde çalışmasını garanti eder.

## 5. CI/CD Süreci ile Entegrasyon

`manifest-mehmetkanus` reposu, `app-mehmetkanus` reposundaki CI/CD pipeline ile sıkı bir entegrasyon içindedir. Bu entegrasyon, uygulamanın otomatik olarak derlenmesini, test edilmesini ve Kubernetes kümelerine dağıtılmasını sağlar.

Başlıca entegrasyon noktaları şunlardır:

*   **GitOps Prensibi**: Bu repo, GitOps prensibinin temelini oluşturur. Uygulama dağıtımları, bu repodaki Kubernetes manifestlerinin durumuna göre yönetilir. ArgoCD aracı, bu repoyu sürekli olarak izler ve herhangi bir değişiklik algıladığında Kubernetes kümesindeki uygulama durumunu günceller.
*   **CI Pipeline Tetikleyicisi**: `app-mehmetkanus` reposundaki CI pipeline, uygulamanın yeni bir versiyonu derlenip konteyner imajı oluşturulduktan sonra bu repoyu günceller. Özellikle, `deploy-to-dev` ve `deploy-to-prod` adımları, ilgili ortamın `kustomization.yaml` dosyasındaki `newTag` değerini güncelleyerek yeni imaj etiketini yazar.

    ```yaml
    # CI/CD pipeline'dan örnek güncelleme adımı
    - name: Update GitOps deployment repo
      run: |
        git clone https://${{ secrets.GITOPS_PAT }}@github.com/${{ env.GITOPS_REPO }} gitops
        cd gitops
        export TAG="${{ github.sha }}"
        sed -i "s|newTag: .*|newTag: ${TAG}|g" ./overlays/dev/kustomization.yaml
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add ./overlays/dev/kustomization.yaml
        git commit -m "Update image:${{ env.APP_IMAGE }} tag to ${TAG}"
        git push origin main
    ```

*   **Otomatik Dağıtım**: `newTag` değeri güncellendiğinde, CD aracı (ArgoCD), bu değişikliği algılar ve Kubernetes kümesindeki ilgili Deployment kaynağını güncelleyerek uygulamanın yeni imajla yeniden dağıtılmasını sağlar. Bu, manuel müdahaleye gerek kalmadan sürekli dağıtım sağlar.
*   **Ortam Ayrımı**: CI/CD pipeline, `dev` ve `prod` ortamları için ayrı dağıtım adımlarına sahiptir. Bu, her ortamın kendi `kustomization.yaml` dosyası ve yamaları aracılığıyla bağımsız olarak yönetilmesini sağlar.

Bu entegrasyon, geliştirme sürecini hızlandırır, dağıtım hatalarını azaltır ve uygulamanın yaşam döngüsünü baştan sona otomatikleştirir.

## 6. Kustomize Gereksinimleri ile Uyum

`manifest-mehmetkanus` reposu, DevOps porjesi tarafından belirtilen Kubernetes manifestleri ile ilgili gereksinimleri tam olarak karşılamaktadır:

*   **Tek K8S cluster’i içerisinde dev ve prod adı ile iki ortam bulunacak, ortamlar birbirinden namespace seviyesinde ayri olacak:**
    *   Repo yapısı, `overlays/dev` ve `overlays/prod` dizinleri aracılığıyla `dev` ve `prod` ortamlarını ayrı ayrı yönetir. Her bir ortamın `kustomization.yaml` dosyasında belirtilen `namespace: dev` veya `namespace: prod` ile ortamlar namespace seviyesinde ayrılmıştır.

*   **dev ve prod ortamlarının K8S manifestleri tek bir git reposunda (“manifest-[aday_ismi]”) olacak ve tek branch (master/main) kullanilacak, ortam ozelinde degisen konfigurasyonlar birbirinden kustomization.yaml’lar ile ayrılacak:**
    *   Tüm `dev` ve `prod` ortamlarına ait Kubernetes manifestleri (`base` ve `overlays` dizinleri altında) tek bir Git reposunda (`manifest-mehmetkanus`) bulunmaktadır. Kustomize, `base` manifestlerini kullanarak ve `overlays` dizinlerindeki `kustomization.yaml` dosyaları aracılığıyla ortama özel değişiklikleri (yamalar) uygulayarak bu gereksinimi karşılar.

*   **CI aracı tarafından K8S manifestlerinin olduğu repoya gerekli şekilde push yapılarak son oluşturulan imajın CD aracı tarafından dev konfigürasyonu ile dev ortamına deploy edilmesi tetiklenecek:**
    *   CI/CD pipeline, `app-mehmetkanus` reposundan yeni bir imaj oluşturulduğunda, `manifest-mehmetkanus` reposundaki `overlays/dev/kustomization.yaml` dosyasındaki `newTag` değerini günceller ve bu değişikliği Git reposuna `push` eder. Bu `push` işlemi, ArgoCD aracı tarafından algılanarak `dev` ortamına otomatik dağıtımı tetikler.

*   **CI aracı tarafından K8S manifestlerinin olduğu repoya gerekli şekilde push yapılarak son oluşturulan imajın CD araci tarafından prod konfigürasyonu ile prod ortamına deploy edilmesi tetiklenecek:**
    *   Benzer şekilde, CI/CD pipeline, üretim ortamı için onay alındıktan sonra `overlays/prod/kustomization.yaml` dosyasındaki `newTag` değerini günceller ve Git reposuna `push` eder. Bu da `prod` ortamına otomatik dağıtımı tetikler.

*   **Tüm secretlar Vault’da bulunacak, secret’in kendisi hiç bir şekilde Git’e pushlanmayacak ancak manifest’i versiyon kontrolüne tabi olacaktır. Vault  K8S baglantisi icin Sealed Secret Operator, External Secret Operator vb. araçlar kullanılacak:**
    *   `manifest-mehmetkanus` reposu, hassas verileri doğrudan içermez. Bunun yerine, `base/external-secret.yaml` ve `overlays/*/external-secret-patch.yaml` dosyaları aracılığıyla External Secrets Operator kullanılır. Bu manifestler, sırların HashiCorp Vault gibi harici bir sır yönetim sisteminden güvenli bir şekilde çekilmesini sağlar. Böylece sırlar Git reposunda saklanmazken, sırların nasıl çekileceğine dair manifestler versiyon kontrolü altında tutulur.

Bu repo, DevOps projesi tarafından belirlenen tüm manifest ve dağıtım gereksinimlerini eksiksiz bir şekilde karşılamaktadır.