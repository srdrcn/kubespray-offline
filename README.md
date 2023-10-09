
# Kubespray Offline Kurulum

## Nedir ?


Bu, [Kubespray offline environment](https://kubespray.io/#/docs/offline-environment) için offline kurulum dosyalarıdır.

Bu, şunları destekler:

* Çevrimdışı dosyalarını indir.
    - İşletim sistemine ait Yum/Deb repo dosyalarını indir.
    - Kubespray tarafından kullanılan tüm container image'larını indir.
    - Kubespray için PyPI mirror'larını indir.
    - İndirilen dosyadan containerd'yi yükle.
    - Yum/Deb deposu ve PyPI mirror'ını sağlamak için web sunucusu olarak nginx container'ını başlat.
    - Docker private registry'i başlat.
    - Tüm container image'larını yükle ve bunları private registery'e gönder.

## Gereksinimler

- RHEL 8 / AlmaLinux 8
- Ubuntu 20.04 / 22.04

## Offline gereksinim dosyaları indir

Çevrimdışı dosyaları indirmeden önce, `config.sh` içindeki yapılandırmaları kontrol edin ve düzenleyin.

Eğer container runtime (docker veya containerd) yoksa, önce onu kurun.

* Containerd  için
    -  containerd ve nerdctl'yi kurmak için `install-containerd.sh` komutunu çalıştırın.
    `config.sh` içinde `docker` ortam değişkenini `/usr/local/bin/nerdctl` olarak ayarlayın.


Daha sonra, tüm dosyaları indir:

    $ ./download-all.sh

Tüm artifact'lar  `./outputs` dizininde saklanır.

Bu script, aşağıdaki tüm script dosyalarını çağırır.

* prepare-pkgs.sh
    - Python kur vs.
* prepare-py.sh
    - Python venv ayarlayın, gerekli python paketlerini yükleyin.
* get-kubespray.sh
    - KUBESPRAY_DIR mevcut değilse, kubespray'i indirin ve çıkarın.
* pypi-mirror.sh
    - PyPI mirror dosyalarını indirin.
* download-kubespray-files.sh
    - Kubespray çevrimdışı dosyalarını (container, dosyalar, vb.) indir.
* download-additional-containers.sh
    - Ek konteynerleri indir.
    - İstediğiniz konteyner görüntü repoTag'ını imagelists/*.txt'ye ekleyebilirsiniz.
* create-repo.sh
    - RPM veya DEB repolarını indir.
* copy-target-scripts.sh
    - Hedef düğüm için komut dosyalarını kopyala.

## Target node support scripts

`outputs` dizinindeki tüm içerikleri ansible'ı çalıştıran hedef düğüme kopyala.
Daha sonra `outputs` dizininde aşağıdaki komut dosyalarını çalıştır.

* setup-container.sh
    - Local dosyalardan containerd'yi yükle.
    - Nginx ve registry görüntülerini containerd'ye yükle.
* start-nginx.sh
    - Nginx container'ını başlat.
* setup-offline.sh
    - Local nginx sunucusunu kullanmak için yum/deb repo yapılandırmasını ve PyPI mirror yapılandırmasını ayarla.
* setup-py.sh
    - Local repodan python3 ve venv'i yükle.
* start-registry.sh
    -  docker private registry'yi başlat.
* load-push-images.sh
    - Tüm container image'larını containerd'ye yükle.
    - Tag ve push işlemlerini yap.
* extract-kubespray.sh
    - Kubespray tarball'ını çıkarın ve tüm patch'leri uygulayın.

Nginx ve docker registry portlarını config.sh içinde yapılandırabilirsiniz.


## Kubespray kullanarak kubernetes'i dağıtın

### Gerekli paketleri yükleyin

Venv oluşturun ve etkinleştirin:


    # Örnek
    $ python3 -m venv ~/.venv/default
    $ source ~/.venv/default/bin/activate

Not: Ubuntu 20.04 ve RHEL/CentOS 8 için python 3.9'u kullanmanız gerekir.

    
    # Örnek
    $ python3.9 -m venv ~/.venv/default
    $ source ~/.venv/default/bin/activate

Kubespray'i çıkarın ve patch'leri uygulayın:


    $ ./extract-kubespray.sh
    $ cd kubespray-{version}

Ubuntu 22.04 için, bazı python paketlerini derlemek için derleme paketlerine ihtiyacınız var.


    $ sudo apt install gcc python3-dev libffi-dev libssl-dev

Ansible'ı yükleyin:

    $ pip install -U pip                # update pip
    $ pip install -r requirements.txt   # Install ansible

### offline.yml oluşturun

offline.yml dosyasını oluşturun ve envanter dizininizdeki group_vars/all/offline.yml dizinine yerleştirin.


YOUR_HOSTu registry/nginx  IP'nizle değiştirmeniz gerekir.

```yaml
http_server: "http://YOUR_HOST"
registry_host: "YOUR_HOST:35000"

# Insecure registries for containerd
containerd_registries_mirrors:
  - prefix: "{{ registry_host }}"
    mirrors:
      - host: "http://{{ registry_host }}"
        capabilities: ["pull", "resolve"]
        skip_verify: true

files_repo: "{{ http_server }}/files"
yum_repo: "{{ http_server }}/rpms"
ubuntu_repo: "{{ http_server }}/debs"

# Registry overrides
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"

# Download URLs: See roles/download/defaults/main.yml of kubespray.
kubeadm_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubelet"
# etcd is optional if you **DON'T** use etcd_deployment=host
etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-{{ etcd_version }}-linux-amd64.tar.gz"
cni_download_url: "{{ files_repo }}/kubernetes/cni/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
crictl_download_url: "{{ files_repo }}/kubernetes/cri-tools/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
# If using Calico
calicoctl_download_url: "{{ files_repo }}/kubernetes/calico/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
# If using Calico with kdd
calico_crds_download_url: "{{ files_repo }}/kubernetes/calico/{{ calico_version }}.tar.gz"

runc_download_url: "{{ files_repo }}/runc/{{ runc_version }}/runc.{{ image_arch }}"
nerdctl_download_url: "{{ files_repo }}/nerdctl-{{ nerdctl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
containerd_download_url: "{{ files_repo }}/containerd-{{ containerd_version }}-linux-{{ image_arch }}.tar.gz"
```



### Çevrimdışı repo yapılandırmalarını dağıtın

Ansible kullanarak çevrimdışı repo yapılandırmalarını, yum_repo/ubuntu_repo'nuzu kullanan tüm hedef düğümlere dağıtın.

Önce, çevrimdışı kurulum playbook'unu kubespray dizinine kopyalayın.

    $ cp -r ${outputs_dir}/playbook ${kubespray_dir}

Daha sonra `offline-repo.yml` playbook çalıştırın.



    $ cd ${kubespray_dir}
    $ ansible-playbook -i ${your_inventory_file} offline-repo.yml

### Kubespray'i çalıştırın


Run kubespray ansible playbook.

    # Örnek  
    $ ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
