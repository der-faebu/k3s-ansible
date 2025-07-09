# General
All deployments aim to use the bjw-s/app-template helm chart. This allows us to flexibly deploy a variety of apps.

## Namespace Mediacenter
These deployments can be found in the sub-folder 'mediacenter'

### Arr-Suite

This category includes the arr-applications such as
- sonarr
- radarr
- bazarr
- prowlarr
- lidarr

Apart from this playback, download and requesting apps are installed:
- ombi
- sabnzb
- jellyfin
- audiobookshelf

#### Radarr

```
helm install radarr bjw-s/app-template -f deployments/mediacenter/helm/1_mediacenter/sonarr/values.yaml
````

#### Sonarr

```
helm install radarr bjw-s/app-template  -f ./deployments/mediacenter/helm/radarr/values.yml
````

#### Lidarr

```
helm instaHll lidarr bjw-s/app-template  -f ./deployments/mediacenter/helm/lidarr/values.yml
````

#### Radarr

```
helm install bazarr bjw-s/app-template  -f ./deployments/mediacenter/helm/bazarr/values.yml
````

#### Prowlarr

```
helm install prowlarr bjw-s/app-template  -f ./deployments/mediacenter/helm/bazarr/values.yml
````

#### Ombi
Backups are not necessary as all data is in the pg db.

```
helm install ombi bjw-s/app-template  -f ./deployments/mediacenter/helm/ombi/values.yml
````

#### Autiobookshelf
Deployment hasn't been migrated to new bjw-s charts
```
 helm install audiobookshelf k8s-home-lab/audiobookshelf -f ./deployments/mediacenter/helm/1_mediacenter/audiobookshelf/values.yaml
```

#### sabnzb
````
helm install sabnzb bjw-s/app-template  -f ./deployments/mediacenter/helm/sabnzb/values.yml
``` 
#### 

#### jellyfin
helm install jellyfin bjw-s/app-template -f deployments/mediacenter/helm/1_mediacenter/jellyfin/values.yaml
####


### others
#### Mealie
Requires a secret for the pg access

```
kubectl create secret generic mealie-pg-pw \
    --from-literal=mealie-pg-pw='<PG user password>' \
    --namespace mediacenter \
    --type opaque

helm install mealie bjw-s/app-template -f deployments/mediacenter/helm/3_others/mealie/values.yaml
```