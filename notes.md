"components": {
    "securitySchemes": {
      "bearerAuth": {
        "type": "apiKey",
        "name": "Authorization",
        "in": "header",
        "description": "Entrez votre token avec le préfixe 'Bearer '. Exemple: Bearer eyJhbGc..."
      }
    }
  },
  "security": [
    {
      "bearerAuth": []
    }
  ],






trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  imageRepository: 'monimage'
  containerRegistry: 'myregistry.azurecr.io'
  tag: '$(Build.BuildId)'
  helmRepoUrl: 'https://github.com/votre-org/votre-repo-helm.git'
  helmRepoBranch: 'main'

stages:
- stage: BuildAndUpdate
  jobs:
  - job: BuildPushAndUpdateHelm
    steps:
    # Build de l'image
    - task: Docker@2
      displayName: 'Build and Push Docker image'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: '**/Dockerfile'
        containerRegistry: $(containerRegistry)
        tags: $(tag)

    # Clone manuel du repo Helm
    - task: Bash@3
      displayName: 'Clone Helm repository'
      inputs:
        targetType: 'inline'
        script: |
          git clone --branch $(helmRepoBranch) $(helmRepoUrl) helm-repo
          cd helm-repo
          ls -la

    # Mise à jour du tag
    - task: Bash@3
      displayName: 'Update Helm values'
      inputs:
        targetType: 'inline'
        workingDirectory: 'helm-repo'
        script: |
          echo "Current tag:"
          grep "tag:" values.yaml
          
          # Mise à jour du tag
          sed -i "s/tag: .*/tag: \"$(tag)\"/" values.yaml
          
          echo "New tag:"
          grep "tag:" values.yaml

    # Commit et Push
    - task: Bash@3
      displayName: 'Commit and Push to Helm repo'
      inputs:
        targetType: 'inline'
        workingDirectory: 'helm-repo'
        script: |
          git config user.email "pipeline@company.com"
          git config user.name "Azure Pipeline"
          
          git add values.yaml
          git commit -m "ci: update image tag to $(tag) [skip ci]"
          
          # Push avec PAT
          git push https://$(HELM_PAT)@github.com/votre-org/votre-repo-helm.git HEAD:$(helmRepoBranch)
      env:
        HELM_PAT: $(HelmRepoPAT)
