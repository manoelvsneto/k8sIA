# k8sIA

## Deployments

- [Ollama com LFM3-1.7B em OKE ARM64](k8s/ollama-lfm3-oci-arm64/README.md)

## Azure DevOps

O pipeline [`azure-pipelines.yml`](azure-pipelines.yml) aplica os manifests de Ollama no OKE quando há alterações na branch `main`. Ele usa a conexão de serviço Kubernetes `K8SOracleCloud` e o pool de agentes `Arm64` já configurados no Azure DevOps. O agente precisa alcançar o endpoint da API do cluster; os nós do OKE precisam de acesso ao Docker Hub e ao Hugging Face para obter a imagem e o modelo.

Configure a conexão de serviço `K8SOracleCloud` para o cluster correto e autorize o pipeline a usá-la. Crie também a variável `AZURE_REPOS_OLLAMA_TOKEN` nas variáveis do pipeline, marque-a como secreta e autorize o pipeline a usá-la. O pipeline usa esse valor para criar/atualizar o Secret `azure-repos-ollama` no namespace `argocd`; a credencial não deve ser armazenada em arquivos do repositório. A identidade da conexão precisa criar esse Secret no namespace `argocd`, recursos no namespace `ollama` e a StorageClass `ollama-oci-bv-retain` no escopo do cluster. O pool `Arm64` precisa existir e ter um agente disponível.

O pipeline não compila uma imagem customizada: ele aplica os arquivos YAML diretamente (compatível com o `kubectl` do agente), aguarda o Deployment ficar pronto e aguarda o Job baixar o GGUF do LFM3 para o volume persistente. O endpoint da API permanece interno ao cluster (`ClusterIP`).
