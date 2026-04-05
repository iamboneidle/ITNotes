___
___
# Tags
#kubernetes 
___
# Содержание
- [[#1. Введение]]
- [[#2. Определение доступных типов хранилищ с помощью ресурсов StorageClass]]
- [[#3. Запрос на StorageClass в PVC]]
- [[#4. Принудительная привязка заявки PersistentVolumeClaim к одному из заранее зарезервированных ресурсов PersistentVolume]]
- [[#Литература]]
___
# 1. Введение
Администратор кластера, вместо того чтобы создавать ресурсы PersistentVolume, может развернуть поставщика (provisioner) ресурса PersistentVolume и определить один или более объектов StorageClass, чтобы позволить пользователям выбрать, какой тип ресурса PersistentVolume они хотят. Пользователи могут ссылаться на класс хранилища StorageClass в своих заявках PersistentVolumeClaim, и поставщик будет принимать это во внимание при резервировании постоянного хранилища.
___
# 2. Определение доступных типов хранилищ с помощью ресурсов StorageClass
Прежде чем пользователь сможет создать заявку PersistentVolumeClaim, которая приведет к резервированию нового ресурса PersistentVolume, администратору необходимо создать один или несколько ресурсов StorageClass.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd # |<- плагин для резервирования
parameters:            # |<-
  type: pd-ssd         # | парамтетры. передаваемые поставщику
  zone: europe-west1-b # |__
```
Ресурс StorageClass указывает, какой поставщик (provisioner) должен использоваться для создания ресурса PersistentVolume, когда заявка PersistentVolumeClaim запрашивает этот ресурс StorageClass. Параметры, определенные в определении StorageClass, передаются поставщику и специфичны для каждого плагина поставщика.
___
# 3. Запрос на StorageClass в PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes:
    – ReadWriteOnce
```
Помимо указания размера и режимов доступа, заявка PersistentVolumeClaim теперь также указывает на класс хранилища, который вы хотите использовать. Когда вы создаете заявку, ресурс PersistentVolume образуется поставщиком, ссылка на который указана в ресурсе StorageClass `fast`. Поставщик используется, даже если существующий зарезервированный вручную ресурс PersistentVolume совпадает с заявкой PersistentVolumeClaim.
___
# 4. Принудительная привязка заявки PersistentVolumeClaim к одному из заранее зарезервированных ресурсов PersistentVolume
```yaml
kind: PersistentVolumeClaim
spec:
  storageClassName: ""
```
Указание пустой строки в качестве имени класса хранилища обеспечивает привязку заявки PVC к предварительно зарезервированному ресурсу PV вместо динамического резервирования нового ресурса.
Если бы вы не присвоили атрибуту storageClassName пустую строку, то динамический поставщик томов зарезервировал бы новый ресурс PersistentVolume, несмотря на наличие соответствующего предварительно зарезервированного ресурса PersistentVolume.
___
# Литература
- Марко Лукша – Kubernetes в действии
