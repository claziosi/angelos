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

stages:
- stage: BuildAndUpdate
  jobs:
  - job: BuildPushAndUpdateHelm
    steps:
    # Build de l'image Docker
    - task: Docker@2
      displayName: 'Build and Push Docker image'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: '**/Dockerfile'
        containerRegistry: $(containerRegistry)
        tags: $(tag)

    # Checkout du repo Helm avec credentials
    - checkout: git://VotreProjet/VotreRepoHelm
      displayName: 'Clone Helm repository'
      persistCredentials: true  # IMPORTANT: active les credentials

    # Mise à jour du tag
    - task: Bash@3
      displayName: 'Update Helm values'
      inputs:
        targetType: 'inline'
        script: |
          cd $(Build.SourcesDirectory)
          echo "Current directory:"
          pwd
          ls -la
          
          echo "Updating tag to: $(tag)"
          sed -i "s/tag: .*/tag: \"$(tag)\"/" values.yaml
          
          echo "Updated values:"
          grep "tag:" values.yaml

    # Commit et Push
    - task: Bash@3
      displayName: 'Commit and Push changes'
      inputs:
        targetType: 'inline'
        script: |
          cd $(Build.SourcesDirectory)
          
          git config user.email "pipeline@company.com"
          git config user.name "Azure Pipeline"
          
          git add values.yaml
          git commit -m "ci: update image tag to $(tag) [skip ci]"
          git push origin HEAD:main
