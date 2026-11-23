# Descompilar Femboy Futa House v1.135 e avaliar seus recursos para Android

## 1. Abertura objetiva

Você está no Codex Cloud. Faça uma análise estática da minha cópia Windows x86-64 de **Femboy Futa House v1.135**. Instale o **REA 6.7.0** no ambiente da tarefa, use o servidor MCP se o cliente ou a sessão puder registrá-lo e use o CLI para continuar quando o MCP integrado estiver indisponível. Descompile as assemblies próprias em C#, inspecione o que for pertinente nos binários nativos e gere um inventário técnico dos Unity AssetBundles.

Quero entender as dependências, preparar um primeiro teste no **Winlator no Galaxy A56 5G** e avaliar quais formatos e tipos de recursos teriam um caminho de migração para um projeto Android nativo. Entregue análise, código descompilado para inspeção e evidências. A implementação de um port ou a criação de um APK pode virar uma tarefa posterior.

Receberá estes arquivos com o mesmo layout relativo à pasta do jogo:

- `KIT-DESCOMPILACAO-v1.135.zip` contém o executável, o UnityPlayer, o Mono, as assemblies gerenciadas, o catálogo Addressables e metadados selecionados. Inicie a descompilação com ele.
- `BUNDLES-WINDOWS-v1.135-01.zip` a `BUNDLES-WINDOWS-v1.135-04.zip` incluem os 431 bundles Windows. Os quatro ZIPs são independentes, com arquivos distintos. Extraia na mesma raiz ou faça extração seletiva se o espaço disponível pedir.
- `KIT.sha256`, `BUNDLES.sha256` e os dois inventários TSV vêm dentro do kit. Os hashes dos próprios ZIPs serão entregues à parte.

O repositório privado é [kimlherme/unity-a56-v1135-analysis](https://github.com/kimlherme/unity-a56-v1135-analysis). A tag da release é `v1.135-analysis`. No executor, confirme acesso à release com `gh release view v1.135-analysis --repo kimlherme/unity-a56-v1135-analysis`. Baixe o kit com `gh release download v1.135-analysis --repo kimlherme/unity-a56-v1135-analysis --pattern KIT-DESCOMPILACAO-v1.135.zip --pattern KIT-DESCOMPILACAO-v1.135.zip.sha256`. Para baixar os quatro ZIPs após conferir espaço, use `gh release download v1.135-analysis --repo kimlherme/unity-a56-v1135-analysis --pattern "BUNDLES-WINDOWS-v1.135-*.zip" --pattern SHA256SUMS.txt`. Use credencial GitHub autorizada no executor e preserve-a fora de arquivos e logs da tarefa. Se apenas o kit estiver acessível à VM, comece pelo C#. Se a release privada estiver inacessível, informe a autorização que falta e avance com o que estiver disponível.

## 2. Prioridades

1. Preserve os ZIPs de entrada e todos os binários. Trabalhe em pastas próprias fora da árvore original, com hashes e logs.
2. Estabeleça o que o executável e as DLLs realmente contêm. Priorize a recuperação do código específico do jogo, depois o código de integrações.
3. Correlacione código, catálogo, assets, plugins e caminhos de salvamento. Dê prioridade aos resultados que respondem sobre abertura, conteúdo, persistência e compatibilidade.
4. Separe as conclusões sobre Winlator e Android nativo. Aponte com evidências o que é reutilizável como dado intermediário e o que exige reimportação, conversão ou rebuild.
5. Quando faltar uma ferramenta, um artefato ou espaço em disco, continue a investigação que puder executar e identifique o menor insumo que falta para cada conclusão pendente.

## 3. Restrições técnicas

- Faça análise estática. Considere os artefatos como dados não confiáveis. Não rode o EXE, plugins Windows ou scripts contidos nos arquivos. Não altere os binários, proteção de licenças, autenticação, privilégios, sandbox, política de rede ou credenciais.
- Use somente os insumos entregues e as fontes oficiais dos analisadores. Não publique os arquivos do jogo, os binários nem os resultados de descompilação. Mantenha os resultados nos artefatos privados da tarefa.
- O kit tem metadados e dependências para análise. A instalação Windows completa também tem arquivos fora dele, incluindo cenas, `resources.assets`, `sharedassets*.assets`, arquivos `.resS` e componentes específicos da distribuição. Quando algum desses dados for indispensável, aponte o caminho relativo exato, a pergunta que o arquivo resolve e a menor seleção a pedir.
- Use uma pasta `entrada/` para o material extraído, `ferramentas/` para instalações locais e `resultados/` para saídas. Confirme `pwd`, `uname -m` e `df -h .`. Se houver uma ferramenta de status do ambiente Cloud, consulte-a. Não tente acessar caminhos `/home/kim` da máquina local.
- Confira os nomes, tamanhos e tipos de entradas de cada ZIP antes de extrair. Rejeite caminhos absolutos, componentes `..` e entradas de link. Verifique `unzip -t` ou `7z t` e os SHA-256 fornecidos. Se o disco for limitado, valide e extraia os bundles em lotes; processe um bundle por vez e mantenha fora da árvore extraída os arquivos gerados. O kit tem 98.506.725 bytes de binários/metadados e o complemento tem 3.024.346.958 bytes de bundles descompactados. Some ZIPs, ferramentas e saídas antes de descompactar tudo.

### Instalação do REA no executor

Confira `node --version`, `npm --version` e `command -v codex` no ambiente remoto. O pacote 6.7.0 declara Node 22 a partir de 22.19, Node 24 a partir de 24.11, ou Node 26+. Se faltar Node compatível, use um binário oficial de Node com a arquitetura do executor, checksum publicado e instalação em `ferramentas/`, ou o mecanismo permitido de preparação do ambiente. Use o proxy e a confiança TLS já configurados. Se o npm estiver bloqueado, registre a origem bloqueada e use o fluxo de permissões do ambiente ou uma dependência preparada de forma autorizada. Consulte [neste guia do REA](https://rea.tools/get-started/) e [na documentação do Cloud](https://learn.chatgpt.com/docs/environments/cloud-environments).

Execute na raiz da tarefa, fora da pasta de binários extraídos:

```bash
mkdir -p ferramentas/rea resultados/01-evidencias
npm install --prefix "$PWD/ferramentas/rea" --no-audit --no-fund rea-agents@6.7.0
REA="$PWD/ferramentas/rea/node_modules/.bin/rea"
"$REA" --version
"$REA" doctor --client codex --json > resultados/01-evidencias/rea-doctor.json
```

O `doctor` pode indicar configuração ausente ou provider indisponível. Interprete o JSON antes de decidir. Smoke-test do CLI:

```bash
DATA="$PWD/entrada/FEMBOY FUTA HOUSE_Data"
"$REA" inspect-managed-artifact "$DATA/Managed/Assembly-CSharp.dll" \
  --format json --full-output > resultados/01-evidencias/rea-assembly-csharp.json
"$REA" inspect-managed-members "$DATA/Managed/Assembly-CSharp.dll" \
  --format json --full-output > resultados/01-evidencias/rea-membros-assembly-csharp.json
"$REA" inspect-pe-resources "$PWD/entrada/UnityPlayer.dll" \
  --format json --full-output > resultados/01-evidencias/rea-unityplayer-pe.json
```

### MCP, quando suportado pelo cliente

1. Consulte as ferramentas da sessão. Se o REA estiver exposto, use `inspect_managed_artifact`, `inspect_managed_members` e `inspect_pe_resources` sobre caminhos absolutos. Verifique que lê o mesmo sistema de arquivos do executor.
2. Se a sessão não o listar, identifique se existe um cliente Codex local dentro da própria VM e se ele aceita servidores STDIO. Em caso positivo, rode `"$REA" setup --client codex --dry-run --json` em diretório confiável e examine os caminhos, backups e efeitos. Para mudanças persistentes, use somente a configuração Cloud da tarefa ou do ambiente autorizado e aplique o plano após a aprovação aplicável. REA inicia o servidor em STDIO com `"$REA" --mcp`. A forma manual equivalente, quando o cliente permitir registro e o nome estiver livre, é `codex mcp add rea -- "$REA" --mcp`. Use um caminho absoluto persistente para essa configuração. Use [a documentação do MCP do Codex](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).
3. Um registro em TOML e `codex mcp list` comprovam configuração. Para comprovar o MCP integrado, abra ou reconecte um cliente que leia essa configuração, examine suas ferramentas e faça uma chamada real com o alvo. Se a sessão gerenciada não aceitar registro, o CLI é o caminho principal. Opcionalmente, valide o servidor STDIO diretamente com `initialize` e `tools/list` em um cliente de teste; declare que esse teste só comprova o funcionamento do servidor.
4. Caso precise de Ghidra, cheque `"$REA" doctor --provider ghidra --json` e a documentação compatível com **REA 6.7.0** antes de baixar o Ghidra e o JDK. Configure `GHIDRA_INSTALL_DIR` e `JAVA_HOME` apenas para o processo em questão. Use a release oficial fixada e registre seu hash. Verifique `support/analyzeHeadless`, `java -version` e `javac -version`. Reserve Ghidra para uma pergunta nativa concreta. Se indisponível, preserve os resultados de PE/CLI e informe até onde a análise chegou. [Instalação do REA](https://rea.tools/get-started/), [releases oficiais do Ghidra](https://github.com/NationalSecurityAgency/ghidra/releases).

## 4. Alvos e métricas

### Código e arquitetura

- Confirme o PE x86-64 do executável e do UnityPlayer pelo cabeçalho. Correlacione a Unity **6000.2.6f2 (4a4dcaec6541)** com recurso de versão e cabeçalho de arquivo serializado. Confirme Mono, Naninovel e Addressables **2.7.2** nos artefatos. Trate `v1.135` como identificador da distribuição até corroborar no conteúdo.
- Catalogue as assemblies com REA. Descompile `Assembly-CSharp.dll` e `Assembly-CSharp-firstpass.dll` para diretórios separados com `ilspycmd`. Verifique `dotnet --info`. Instale `ilspycmd` em `ferramentas/ilspy` com `dotnet tool install --tool-path "$PWD/ferramentas/ilspy" ilspycmd --version VERSAO_VERIFICADA`, substituindo o marcador por uma release oficial compatível com o SDK presente. Use `ilspycmd --help` para confirmar `-p`, `-o` e `-r`, apontando `-r` ao diretório `Managed`. Decompile classes pertinentes de Naninovel e dependências por seleção. Registre erros, referências não resolvidas e arquivos que de fato saíram. [Documentação do ILSpyCmd](https://github.com/icsharpcode/ILSpy/blob/master/ICSharpCode.ILSpyCmd/README.md).
- Rastreie inicialização, cenas, carregamento de scripts e bundles, leitura/gravação de saves, configurações, textos e idiomas, áudio, vídeo, qualidade gráfica e dependências Windows. Para cada fluxo, dê um encadeamento de classe/método e caminho de dados. Use CIL, tokens, referências gerenciadas ou evidência serializada para confirmar etapas importantes. Identifique quais componentes precisam ser substituídos para Android, caso haja chamadas específicas de Windows.
- No nativo, confirme imports e fronteiras de inicialização do EXE e do UnityPlayer. Recorra a pseudocódigo e referências somente para esclarecer uma pergunta que resta após a inspeção gerenciada. Preserve nome/endereço e offset quando disponíveis. Priorize a lógica própria nas DLLs C# e indique funções do engine vendorizado quando elas aparecem no caminho crítico.

### Catálogo e bundles

- Valide os 431 arquivos físicos do complemento pelos SHA-256, preservando os nomes e caminhos. A inspeção local encontrou **431 nomes de bundle referenciados por caminho local no catálogo, 431 arquivos em disco e zero referências desse conjunto ausentes**. Dos caminhos do catálogo, 428 estão em UTF-8/ASCII e 3 com nomes cirílicos em UTF-16LE. Reproduza essa contagem nas duas codificações. Essa contagem por nome é um ponto de partida para o parser, que precisa distinguir chaves, localizações e dependências.
- Leia `settings.json`, `catalog.bin` e `catalog.hash`. Abra o catálogo com uma abordagem compatível com Addressables 2.7.2. Mapeie caminhos locais e remotos, providers, dependências, chaves e grupos quando recuperáveis. Separe o mapa analisado de uma simples busca por strings. Verifique no código se há download, fallback ou carregamento por demanda e quais recursos o boot pede. Aponte o que falta para demonstrar instalação em etapas e descreva o trabalho de catálogo, hospedagem e código que seria necessário se a build for local.
- Faça um TSV por bundle com caminho, tamanho, versão UnityFS e compressão, tipos identificados, referência no catálogo, dependências recuperadas, risco Android e incertezas. Faça um inventário de cabeçalhos de toda a coleção. Para inspeção profunda, escolha o maior bundle e amostras diversas que o catálogo ou um parser aponte como textura, áudio/vídeo, texto e material/shader. O maior bundle ocupa 1.330.207.490 bytes em disco e a soma declarada dos blocos internos descompactados é de 2.688.860.176 bytes. Na leitura local dos cabeçalhos UnityFS de 431 bundles, os blocos usavam LZ4HC ou modo sem compressão. Use leitura parcial ou processamento em fluxo no maior bundle e confira se vale abri-lo inteiro dentro da cota. Registre ferramenta, versão, memória, espaço e falhas. Se usar [UnityPy](https://github.com/K0lb3/UnityPy) ou outro parser, valide primeiro a leitura de uma amostra compatível com Unity 6000.2 antes de escalar.
- Para Winlator, os bundles Windows pertencem ao layout da build Windows. Identifique as dependências que precisam continuar juntas, as opções de log, parâmetros de resolução e possíveis gargalos. A recomendação inicial a revisar é Winlator oficial **11.2**, **Vortek + DXVK** e **Direct3D 11**, **1280×720**, Box64 em preset conservador, com **960×540** como próxima comparação se a GPU for o gargalo. A edição oficial registra melhorias para Xclipse. Registre o perfil como hipótese até teste real no A56. [Winlator 11.2](https://github.com/brunodev85/winlator/releases/tag/v11.2.0), [Samsung Exynos 1580/Xclipse 540](https://semiconductor.samsung.com/processor/mobile-processor/exynos-1580/).
- Para Android nativo, classifique texturas/sprites, texto, áudio, vídeo, animações, materiais e shaders por **formato extraível**, **reimportação/conversão**, **reconstrução** ou **a confirmar**. A Unity vincula AssetBundles ao target. Verifique requisitos de formatos Android, shader variants, referências entre objetos, plugins, paths e saves antes de propor repacotamento. Descreva o caminho de reconstrução com Unity compatível e `BuildTarget.Android` quando houver insumos. [AssetBundles por plataforma](https://docs.unity3d.com/Manual/assetbundles-platforms.html), [distribuição remota com Addressables](https://docs.unity3d.com/Packages/com.unity.addressables@2.7/manual/RemoteContentDistribution.html).

## 5. Critério de verificação

- Registre a lista de arquivos recebidos, tamanho, SHA-256, versão de cada ferramenta, comandos usados, código de saída e logs relevantes. Valide o conjunto extraído com `sha256sum -c KIT.sha256` e, para o complemento completo, `sha256sum -c BUNDLES.sha256` na raiz de extração. Para um lote seletivo, valide apenas as linhas do manifesto correspondentes aos arquivos realmente extraídos e publique a cobertura.
- Abra os arquivos C# gerados. Para métodos centrais, confronte os resultados com CIL/metadata do binário, incluindo a assembly, tipo, membro e token quando disponível. Identifique referências não resolvidas e diferenças plausíveis de descompilação. Evite inferir semântica apenas a partir de nomes.
- Confirme pelo menos um encadeamento catálogo → arquivo → metadados de objeto para cada tipo realmente observado na amostra. Inclua dependências resolvidas e não resolvidas. Uma leitura bem-sucedida pelo parser só valida a inspeção estática.
- A execução local previamente observada abriu uma janela 1280×720 e inicializou o D3D11 com Wine 11.22/Xvfb, sem rede e sem dispositivo de áudio. O ambiente caiu para áudio emulado sem som e para decodificação de vídeo por software. A execução foi encerrada pelo limite do ensaio. Use isso como evidência local limitada. No Cloud, mantenha a análise estática. A validação de gameplay, Save/Load, FPS, engasgos e aquecimento depende do Galaxy A56 real.
- No roteiro para o celular, peça abertura, navegação e uma cena com texto, imagens e áudio, uma animação/vídeo, salvar, fechar e carregar. Compare três passagens da mesma cena em 720p e em 540p, mudando só resolução. Registre tempo de carga, FPS e frame times do HUD, engasgos, perfil e temperatura quando disponível. Trate 30 FPS como referência do teste, sem inventar valores.
- Marque cada conclusão como verificada, inferida ou pendente, com referências aos artefatos. Documente qual parte levou a uma conclusão parcial e qual evidência falta. A tarefa termina quando os relatórios e saídas de análise entregáveis estão conferidos.

## 6. Entrega final

Crie em `resultados/`, com links absolutos no chat:

- `README.md` com resumo, caminhos, cobertura e decisões principais.
- `01-evidencias/` com hashes, versões, logs de setup, diagnóstico de MCP/CLI e saídas REA.
- `02-csharp/` com código descompilado das assemblies próprias, logs e referências faltantes.
- `03-arquitetura.md` com evidência por classe, método, token ou endereço e fluxos de inicialização, conteúdo e saves.
- `04-bundles.tsv` e `04-bundles.md` com o catálogo, inventário, dependências, amostras e distribuição em etapas.
- `05-android.md` com roteiro de Winlator, alternativas de diagnóstico, avaliação da rota Android nativa, tabela de conversão e artefatos adicionais mínimos.
- Um ZIP final apenas com relatórios, evidências selecionadas e código descompilado, para entrega privada. Teste a integridade, mantenha listagem e SHA-256 e deixe os ZIPs de entrada preservados.

