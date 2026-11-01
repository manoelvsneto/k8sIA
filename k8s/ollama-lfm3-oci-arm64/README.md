# Ollama + LFM3-1.7B em Oracle OKE ARM64

Este diretório instala o servidor Ollama, persiste os modelos em OCI Block Volume e baixa o GGUF `Q4_K_M` de `LiquidAI/LFM3-1.7B-GGUF` automaticamente. O deployment fixa os pods em nós ARM64 e deixa a API como `ClusterIP`, sem publicar uma API Ollama aberta na internet.

## Pré-requisitos

- Cluster Oracle Kubernetes Engine (OKE) com um node pool ARM64, por exemplo em shape Ampere A1/A2. O deployment usa `kubernetes.io/arch: arm64`; sem um nó ARM64 com recursos livres, o pod ficará `Pending`.
- A imagem oficial `ollama/ollama:0.40.12` publica variante `linux/arm64`; essa versão também atende ao mínimo indicado pelo model card do LFM3.
- Add-on CSI de Block Volume da OCI habilitado. O Kustomize cria a StorageClass `ollama-oci-bv-retain`, que usa o provisioner oficial `blockvolume.csi.oraclecloud.com`.
- Permissão para criar StorageClasses (recurso de cluster), namespaces e demais recursos.
- Saída de rede dos nós para o Docker Hub e para o Hugging Face, incluindo o CDN usado pelo Hugging Face. Para worker nodes em subnet privada, configure uma rota de saída adequada.
- Para o pod Ollama, reserve no nó ao menos 2 OCPU e 4 GiB de memória; os limites configurados são 4 CPU e 8 GiB. Um nó com mais memória deixa margem para o sistema e para contextos maiores.

O PVC pede `50Gi`, pois o Block Volume CSI da OCI arredonda pedidos menores para o mínimo de 50 GB. A StorageClass usa política `Retain`: ao apagar o PVC, o volume OCI permanece e precisa de limpeza manual quando não for mais necessário.

## Aplicar

Na raiz do repositório:

```sh
kubectl get nodes -L kubernetes.io/arch
kubectl get storageclass
kubectl apply -k k8s/ollama-lfm3-oci-arm64
```

### Azure DevOps

O pipeline na raiz do repositório aplica esses mesmos manifests em cada alteração na branch `main`. Configure no Azure DevOps:

- Uma conexão de serviço Kubernetes chamada `K8SOracleCloud`, apontando para o cluster OKE desejado e autorizada para este pipeline.
- Uma variável de pipeline `AZURE_REPOS_OLLAMA_TOKEN`, marcada como secreta e autorizada para uso pelo pipeline. O pipeline a usa para criar/atualizar o Secret de repositório do Argo CD no namespace `argocd`; não grave o token em manifests ou no Git.
- Um pool de agentes chamado `Arm64`, com um agente online capaz de acessar o endpoint da API Kubernetes. Se o endpoint do cluster for privado, o agente precisa estar na rede com acesso a ele.
- Permissões para a identidade da conexão criar Secrets no namespace `argocd`, recursos no namespace `ollama` e a StorageClass `ollama-oci-bv-retain` no escopo do cluster.

O pipeline aplica os manifests, aguarda o Deployment Ollama ficar pronto e espera o Job terminar o download do modelo. Se o Job falhar ou exceder o tempo limite, consulte seus logs com `kubectl -n ollama logs job/ollama-pull-lfm3`.

Acompanhe o servidor, o PVC e o Job que baixa o modelo:

```sh
kubectl -n ollama get pods,pvc,jobs
kubectl -n ollama logs job/ollama-pull-lfm3 -f
```

O download do modelo ocorre uma vez por Job e fica guardado no PVC. Se precisar executar novamente o pull, apague o Job concluído e reaplique os manifests:

```sh
kubectl -n ollama delete job ollama-pull-lfm3
kubectl apply -k k8s/ollama-lfm3-oci-arm64
```

## Acessar a API

O Service é interno ao cluster. Para testar do seu computador sem expor a porta publicamente, mantenha este comando aberto:

```sh
kubectl -n ollama port-forward svc/ollama 11434:11434
```

Em outro terminal Bash, envie uma geração de teste:

```sh
curl http://127.0.0.1:11434/api/generate -d '{
  "model": "hf.co/LiquidAI/LFM3-1.7B-GGUF:Q4_K_M",
  "prompt": "Resuma o Kubernetes em uma frase.",
  "stream": false,
  "options": {"num_ctx": 8192}
}'
```

Aplicações no cluster podem usar `http://ollama.ollama.svc.cluster.local:11434`.

## Ajustes

- O contexto está limitado a 8.192 tokens via `OLLAMA_CONTEXT_LENGTH` para conter o uso de memória em CPU ARM64. Ajuste `OLLAMA_CONTEXT_LENGTH` e os recursos do pod juntos se precisar de mais contexto.
- Para outra quantização, altere `OLLAMA_MODEL` em `model-pull-job.yaml`, usando uma variante publicada no repositório GGUF.
- O modelo oficial é distribuído sob a licença LFM Open License v1.0; revise a licença antes de redistribuir ou incorporar o modelo em um produto.
- Para acesso externo, adicione um gateway com autenticação e TLS. Não publique diretamente o Service Ollama sem uma camada de proteção.

## Referências

- [Modelo LFM3-1.7B original](https://huggingface.co/LiquidAI/LFM3-1.7B)
- [GGUF oficial e instrução de execução com Ollama](https://huggingface.co/LiquidAI/LFM3-1.7B-GGUF)
- [Imagem oficial Ollama 0.40.12 e plataformas](https://hub.docker.com/r/ollama/ollama/tags?name=0.40.12)
- [OKE: executar aplicações em nós ARM](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengrunningarmnodes.htm)
- [OKE: provisionar PVCs em OCI Block Volume](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingpersistentvolumeclaim_topic-Provisioning_PVCs_on_BV.htm)
- [Ollama: definir o tamanho do contexto](https://docs.ollama.com/faq)
