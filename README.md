# Instructions to install Jenkins in k8s
# First of all it's a good practice to create a specific namespace for it, where everything related to Jenkins
# such as pods, services, volumes will be stored.
kubectl create namespace jenkins"

# Next we'll create the deployment yaml file for Jenkins
"nano jenkins.yaml"

Vu qu'il faut que Jenkins cree des pods et les manage via le k8s api, il faut donc donner au pod de Jenkins
le droit de manager des pods k8s, (le pod et le container jenkins vont partager les mêmes permissions)

Et la façon pro de faire ca c est de use les service aacounts. Donc on va modifier notre deployment jenkins
et lui accorder un service account rattaché à la permission clusterRole admin.

1. Creer le serviceaccount : kubectl create serviceaccount jenkins-agent -n jenkins
2. Faire le lien entre le role (il y a des roles predefini et le serviceaccoutn) :
kubectl create rolebinding jenkins-agent-admin \
  --clusterrole=admin \
  --serviceaccount=jenkins:jenkins-agent \
  --namespace=jenkins

  ou bien 
  kubectl create clusterrolebinding jenkins-agent-admin \
  --clusterrole=admin \
  --serviceaccount=jenkins:jenkins-agent pour acceder à tous les namespaces

3. Add this line :       serviceAccountName: jenkins  # ← Ajoute/modifie cette ligne
under the spec in the jenkins deployment

4. Ensuite plus d'info dans le jenkinsfile

5. Creer un secret sur k8s avec les cred de docker Hub car les agents pods sont temporaires, il faut stocker 
l'image resultante quelque part.

Creer le secret par commande (pas par fichier)
---
kubectl create secret docker-registry docker-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=username\
  --docker-password=pwd \
  --docker-email=mail \
  -n jenkins

  docker-registry est un type special de secret dans k8s
  https://index.docker.io/v1/ c'est l'adresse du registry, hub.docker.com c'est juste pour le portail

  6. Monter le secret dans l'agent, comme fichier de configuration


  ---------------------------------
  Etapes pour le ci cd k8s avec Jenkins
  Il s'agira d'utiliser k8s comme plugin pour permettre à Jenkins d'utiliser des pods ephemères de k8s comme agents
  Et par la suite deployer l'app sur le même cluster.

  1. On a commencé par installer Jenkins comme un pod, avec son deployment et son service. On a exposé deux ports
  mais on en avait pas besoin car le supposé port 5000 est legacy.
  Au pod de Jenkins on a grephé un volume. Ca commence par la creation d'un persistance volume claim. Où tu precises la taille et le type de stockage dont tu as besoin. Ensuite tu peux creer un volume correspondant à ce pvc, ou ne rien faire car si k8s voit qu'il n'ya pas de volumes avec ces features, il va en créér un.
  Donc dans le deployment, sous la section containers où tu precises l'image de Jenkins, ajouter une section qui precise e volume qui sera monté sur le pod, en utilisant "VolumeMount" :
  "# Jenkins-as-a-pod" 
