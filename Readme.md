# Distribuição e Auto Update do Tera Printer

## Auditoria e decisões

O app está em `apps/printer-agent`: versão independente 0.1.2, Electron ^43.2.0
(resolvido 43.4.1), Vite 8, TypeScript e pnpm 10.28.1. O empacotamento existente
usa electron-builder 26.15.3, NSIS Windows x64, DMG/ZIP universal macOS e AppImage.
O atualizador usa electron-updater **6.8.9**. appId **com.teragestor.printer**,
productName **Tera Printer**, versão, ícones e opções NSIS existentes foram
preservados nesta alteração. Main/preload são compilados pelo plugin Electron do
Vite; tsc valida tipos. Instância única, tray e autostart `--background` já existem.

A decisão atual é **Windows sem Code Signing**, sem certificado autoassinado,
OV/EV ou serviço de assinatura. macOS continua exigindo Developer ID e notarização.
O workflow anterior exigia CSC_LINK e forceCodeSigning também no Windows; essa
obrigatoriedade foi retirada apenas do job Windows. Publicação usa GitHub Releases
em repositório público de binários, enquanto o código pode ficar privado. GHCR
continua atendendo containers; não é feed de instaladores. Não há infraestrutura
nova, dependências adicionadas ou alterações de schema Prisma nesta etapa.

### Identidade de instalações anteriores

O workspace já continha appId com.teragestor.printer ao iniciar esta etapa, enquanto
HEAD ainda registra com.ccnossasenhoradenazare.pdv. Essa alteração anterior não foi
revertida nem modificada aqui. Antes de atualizar instalações legadas, confirme qual
ID foi utilizado no instalador delas. Não presuma uma migração automática entre IDs
diferentes: o ensaio de atualização deve usar o mesmo appId nas duas versões, ou
ser precedido de uma migração de identidade revisada. Esta compatibilidade está
pendente de validação com o instalador realmente distribuído.

## Build Windows sem certificado

`win.signExecutable: false` é suportado pela versão instalada do builder. Ele
impede assinatura do executável, desinstalador e instalador, mantendo a edição dos
recursos (ícone, versão, metadados e integridade ASAR). Não foi usado
`signAndEditExecutable: false`, que removeria também essa edição.

Não é necessário configurar CSC_LINK, CSC_KEY_PASSWORD, WIN_CSC_LINK ou
WIN_CSC_KEY_PASSWORD. O job Windows usa CSC_IDENTITY_AUTO_DISCOVERY=false e não
recebe certificados dos secrets. O padrão de verificação de assinatura do updater
não foi alterado: não há verifyUpdateCodeSignature=false, publisherName=null
forçado, verificador customizado ou bypass do Windows/Defender/SmartScreen.

Com Node 24 e pnpm 10.28.1, na raiz do monorepo:

```sh
pnpm install --frozen-lockfile --filter=tera-printer...
# Defina PRINTER_RELEASE_OWNER e PRINTER_RELEASE_REPO com o repo público real.
pnpm --filter tera-printer typecheck
pnpm --filter tera-printer test:printing
pnpm --filter tera-printer test:updates
pnpm --filter tera-printer test:distribution
pnpm --filter tera-printer smoke
pnpm --filter tera-printer dist:win --publish never
pnpm --filter tera-printer verify:win
```

Para gerar uma release com Auto Update, o feed deve ser configurado mesmo usando
--publish never: essa opção impede upload, mas mantém geração de metadados.
As duas variáveis do feed são públicas e não são credenciais. Não execute dist:win com forceCodeSigning=true
neste perfil sem assinatura. Saídas do build com feed em `apps/printer-agent/release/`:

- `Tera-Printer-Windows-x64.exe`: instalador NSIS sem Authenticode.
- `Tera-Printer-Windows-x64.exe.blockmap`: atualização diferencial.
- `latest.yml`: versão, nome, tamanho e SHA-512 do instalador.
- `win-unpacked/resources/app-update.yml`: configuração pública do feed.

`verify:win` confere versão, SHA-512 e tamanho, blockmap, feed público e ausência
de assinatura Authenticode nos cabeçalhos PE do app/instalador. Também recusa
configuração que desabilite explicitamente a verificação de assinatura. Esse
comando verifica o perfil **sem assinatura**; não deve ser utilizado para exigir
que uma futura distribuição assinada permaneça sem assinatura.

## Build local sem variáveis de distribuição

O comando abaixo funciona sem PRINTER_RELEASE_OWNER/PRINTER_RELEASE_REPO, fora do CI:

```sh
pnpm --filter tera-printer dist:win --publish never
```

Nesse caso o instalador vai para `apps/printer-agent/release-local/` e **não inclui
feed ou Auto Update**. O tray informa que atualizações não estão configuradas.
Não são usados owner/repo fictícios nem inferidos do git. O diretório separado evita
confundir o instalador local com metadados de releases anteriores. `verify:win` é
para builds de release com feed e não se aplica ao instalador local.

`electron-builder.config.cjs` estende a configuração YAML e resolve o feed antes do
builder. Ambas as variáveis presentes habilitam metadados normalmente. Só uma
variável presente, tentativa de publicação ou CI sem feed falham com orientação.
Nenhum segredo de publicação é necessário para apenas compilar localmente.

Para gerar com Auto Update no PowerShell (substitua pelo repo público real):

```powershell
$env:PRINTER_RELEASE_OWNER = "SEU_OWNER"
$env:PRINTER_RELEASE_REPO = "SEU_REPOSITORIO_DE_BINARIOS"
pnpm --filter tera-printer dist:win --publish never
pnpm --filter tera-printer verify:win
```

Variáveis configuradas no GitHub Actions não são automaticamente exportadas no
terminal local. Se você guardá-las em .env, exporte-as antes do comando: o script
não adiciona uma biblioteca de dotenv nem carrega arquivos secretos automaticamente.

## Compatibilidade e limites do Auto Update sem assinatura

A auditoria do código instalado confirmou: NsisUpdater.verifySignature em
6.8.9 retorna sem validação Authenticode quando app-update.yml não possui
publisherName. O builder 26, sem certificado, gera naturalmente esse perfil.
Não removemos uma identidade configurada nem substituímos o verificador.
O caminho é coberto por teste de contrato executando a biblioteca real instalada.

O download continua verificando SHA-512 contra latest.yml. Versões maiores estáveis
são exigidas; downgrade e prereleases estão desabilitados. O provider público
GitHub resolve os instaladores via HTTPS. NSIS mantém download diferencial e
fallback completo da biblioteca. Metadados inválidos/hash ausente ou download
corrompido são rejeitados pelos mecanismos existentes.

**Não há autenticação criptográfica do editor neste perfil.** SHA-512 detecta
alterações em relação ao manifesto, mas não protege se um invasor conseguir trocar
simultaneamente o manifesto e o arquivo. HTTPS e o controle do repositório/token
de publicação são a fronteira de confiança. Não equivale à segurança de pacotes
assinados. Proteja a conta GitHub, permissões e acesso às releases. Tokens nunca
são incluídos no executável ou no dashboard.

Instaladores sem assinatura podem mostrar editor desconhecido, avisos do
SmartScreen ou ser bloqueados por políticas corporativas. Não contorne esses
bloqueios e não desative antivírus/Defender/SmartScreen. Em ambiente gerenciado,
consulte a política da organização; pode ser necessária distribuição assinada.

Esse comportamento é específico das versões instaladas. electron-updater 7 e
builder 27 possuem requisitos diferentes para publisherName/assinatura. Ao atualizar
essas ferramentas, execute os testes de contrato e homologue novamente. Não altere
flags de verificação para contornar uma futura incompatibilidade.

## Operação e preservação da impressão

UpdateService inicia apenas em Windows/macOS empacotados, após HTTP e renderer
prontos. Verifica no boot, pelo tray e a cada 6 horas. Configure
PRINTER_UPDATE_INTERVAL_MS entre 900000 e 86400000 ms; inválidos usam 6 horas.
Estados: IDLE, CHECKING, AVAILABLE, DOWNLOADING, DOWNLOADED, INSTALLING, ERROR.
Versão, progresso, bytes e velocidade ficam no serviço; eventos relevantes usam o
logger console existente, sem tokens ou logs por bloco. Falhas preservam a versão
atual e permitem nova verificação. O executor tem timeout de inatividade de 60s.

Download não pausa impressão. A instalação exige confirmação pelo tray porque a
fila é em memória e o adaptador confirma envio ao SO, não conclusão física no papel.
PrintService conta desde a entrada, incluindo consultas e espera FIFO. Instalação
só fecha admissão atomicamente com contador zero. Novos trabalhos recebem 503
antes de serem aceitos; adiar/falhar reabre admissão e permite retry do requestId.
Não há encerramento forçado nem instalação automática no quit. Sair também é
adiado com trabalho ativo. O reinício solicitado ao instalador restaura o serviço.

HTTP local autenticado, controle de origem, IPC validado, sandbox,
contextIsolation e nodeIntegration=false permanecem. Nenhuma API de update foi
exposta ao renderer. Settings/token ficam em app.getPath('userData'), fora dos
arquivos substituídos pelo instalador; impressoras permanecem no SO. Não há
migração de dados/configuração. Deduplicação em memória não sobrevive ao reinício;
não reenvie cegamente trabalhos com resultado incerto.

No macOS, a preparação nativa Squirrel continua ocorrendo após confirmação e com
admissão fechada. Erro/timeout de 60s remove listeners temporários antes de liberar
o serviço, evitando solicitação de reinício por evento atrasado.

## GitHub Actions e publicação

No repositório do código, configure Actions Variables:

- PRINTER_RELEASE_OWNER: proprietário do repositório **público** de binários.
- PRINTER_RELEASE_REPO: nome desse repositório, sem URL.

Secret necessário para Windows: PRINTER_RELEASE_TOKEN, token restrito ao repo de
binários com Contents:write. Não são necessários secrets de Code Signing Windows.
O token fica somente nos passos que validam o repo e criam a release.

1. Aumente a versão SemVer em apps/printer-agent/package.json.
2. Envie a tag printer-v<VERSÃO>, ou execute Tera Printer Desktop manualmente.
   O padrão é Windows. Execução manual também permite macos ou both.
3. O pipeline instala com frozen-lockfile, executa tipos/lint/testes/smoke, compila,
   gera NSIS sem certificado e executa verify:win. Só então armazena os artefatos.
4. Após os builds selecionados passarem, cria uma release **draft** completa no
   repo público. Assets existentes não são sobrescritos: use nova versão.
5. Homologue e publique o draft como latest para disponibilizar o feed.
   Nenhuma release real foi publicada durante esta implementação.

Mantenha todas as versões publicadas imutáveis e anteriores disponíveis para
atualização diferencial. Uma release apenas Windows não contém latest-mac.yml e
não oferece atualização Mac. Antes de distribuir Mac, publique versão conjunta
nova escolhendo both. O caminho Windows não depende de secrets/build macOS.

## Dashboard e macOS

Configure NEXT_PUBLIC_PRINT_AGENT_DOWNLOAD_URL_WINDOWS com:
`https://github.com/OWNER/REPO/releases/latest/download/Tera-Printer-Windows-x64.exe`.
NEXT_PUBLIC_PRINT_AGENT_DOWNLOAD_URL é fallback legado somente Windows. Para Mac,
NEXT_PUBLIC_PRINT_AGENT_DOWNLOAD_URL_MACOS aponta para
`https://github.com/OWNER/REPO/releases/latest/download/Tera-Printer-macOS-universal.dmg`.
Essas vars passam pelo Docker/Turbo e exigem rebuild do dashboard. Configurações de
impressão sugerem o SO, permitem troca manual e não oferecem links ausentes/inseguros.

macOS mantém secrets PRINTER_MAC_CSC_LINK, PRINTER_MAC_CSC_KEY_PASSWORD,
PRINTER_APPLE_ID, PRINTER_APPLE_APP_SPECIFIC_PASSWORD e PRINTER_APPLE_TEAM_ID.
Exige Developer ID Application, Hardened Runtime, notarização, verificação codesign,
ticket stapled e Gatekeeper no pipeline. Seu DMG/ZIP universal suporta Intel e
Apple Silicon (macOS 12+). Linux permanece sem updater ativado.

## Teste real obrigatório no Windows

Use um repo público de homologação e duas versões **sem assinatura**, por exemplo
1.0.0 e 1.0.1 (ou duas versões maiores válidas da série atual). Não reutilize tags
ou assets de produção. Gere a primeira com o feed de homologação, instale no Windows
10/11 x64 e publique a segunda no mesmo repo como latest.

- Registre versão, origens, autostart, impressoras e preferências sem expor tokens.
- Confirme detecção/download em background e imprima recibos/NFC-e durante download.
- Com impressão ativa, instalar/sair deve ser adiado. Confira o papel antes de
  autorizar pelo tray. Adie primeiro e verifique que o serviço permanece imprimindo.
- Autorize a atualização e confirme que o app instala/reinicia sem executar outro
  instalador manualmente. Confira nova versão no health/tray e integração do PDV.
- Verifique settings/token/origens/autostart preservados e impressão física real.
- Teste offline, download interrompido, hash inválido e metadados inválidos:
  versão atual permanece operacional. Verifique que versão inferior não é instalada.
- Registre SmartScreen/políticas reais, sem desativar proteções para aprovar o ensaio.

Build e testes com doubles não comprovam instalação/reinício. Este ensaio não foi
executado: o ambiente disponível é macOS, sem sessão Windows/hardware conectado.
Instalações anteriores sem updater precisam instalar a primeira versão com o
recurso uma vez. Se a instalação antiga exige publisherName/assinatura, ela não
deve aceitar uma versão unsigned: não remova essa proteção; mantenha canal assinado
ou realize instalação inicial revisada conforme a política operacional.

## Diagnóstico, contenção e recuperação

Procure `[PrinterAgent] AutoUpdate` no stdout/stderr capturado. ERROR permite retry
manual/periódico. Confira HTTPS, repo público, release publicada/latest, SemVer maior,
assets completos e SHA-512. Há cache de downloads da biblioteca, sem novo banco.

Para conter release problemática, converta em draft/remova sua visibilidade. Isso
não revoga downloads já concluídos: oriente operadores a adiar instalação. Publique
correção com versão maior, sem sobrescrever assets ou habilitar downgrade. Se não
reiniciar, abra pelo atalho. Reinstalação deve preservar appId e userData; não apague
token/settings para corrigir update. Não há rollback automático.

## Assinatura futura

Para retornar ao perfil assinado, habilite win.signExecutable=true e
forceCodeSigning=true no build Windows, mapeando os secrets do método de assinatura
escolhido. Para PFX compatível, CSC_LINK/CSC_KEY_PASSWORD; token/HSM/serviço exige
adaptador apropriado ao fornecedor, sem refatorar o serviço de update.
Substitua o passo verify:win (que exige ausência de assinatura) por verificação de
Authenticode e do publisherName gerado no app-update.yml. O builder deve derivar a
identidade do certificado; não configure publisherName fictício. Homologue a
transição unsigned→signed antes de publicar. Nenhum serviço pago foi integrado.

Referências: [migração v27](https://www.electron.build/docs/migration/v27-build-changes/),
[assinatura Windows v26](https://www.electron.build/v26/docs/features/code-signing/code-signing-win/).

## Validações desta etapa unsigned

Executadas com Node 24.11.1 no macOS ARM64:

- `dist:win --publish never` com CSC_LINK, CSC_KEY_PASSWORD, WIN_CSC_LINK e
  WIN_CSC_KEY_PASSWORD removidos do ambiente, CSC_IDENTITY_AUTO_DISCOVERY=false:
  aprovado. Gerou NSIS x64 de aproximadamente 102 MiB, blockmap e latest.yml.
  Vite/main/preload e tsc concluíram durante o build.
- `verify:win <diretório de validação>`: aprovado; conferiu SHA-512, tamanho,
  versão, blockmap, ausência de Authenticode no app/instalador e configuração do feed.
- `test:distribution`: 5 testes aprovados usando o updater instalado e seus
  mecanismos reais de parsing, SHA-512 e resolução GitHub HTTPS.
- `test:updates`: 7 aprovados; `test:printing`: 6 aprovados; `smoke`: aprovado,
  incluindo pausa HTTP 503 e retry após adiamento.
- `typecheck`: aprovado. `lint`: sucesso com 5 avisos existentes em styles.css.
  Biome check dos arquivos novos e git diff --check aprovados.
- YAML/matrizes do workflow validados. Pré-requisitos Windows executados em teste
  isolado sem certificados passaram; macOS sem certificado continuou sendo recusado.
  A consulta GitHub foi substituída por resposta local nesse teste de pré-requisitos;
  não é evidência de acesso ao repo remoto ou publicação real.

Os artefatos de validação foram gerados em diretório temporário, sem publicar.
O repo origin foi usado como valor local de feed; sua visibilidade e disponibilidade
como canal público não foram verificadas. Configure o repo público aprovado e
reconstrua antes de distribuir. Não houve teste de execução/instalação, impressão
física, preservação de configurações ou atualização real entre versões no Windows.
Esses critérios dependem do ensaio descrito acima e permanecem pendentes.

### Correção do build local sem PRINTER_RELEASE_OWNER

Validados os empacotamentos Windows 0.1.2 em diretórios temporários separados:
sem ambas as variáveis/CI, NSIS gerado sem app-update.yml/latest.yml; com feed
configurado, NSIS/metadados gerados e verify:win aprovado. O feed usado no ensaio
não foi publicado nem confirmado como canal público. Foram aprovados typecheck,
6 testes de distribuição, 7 de update, 6 de impressão e smoke; lint concluiu com os
5 avisos CSS existentes. YAML e Biome check dos arquivos alterados aprovados.
Os filtros dos workflows foram atualizados de teragestor-pos para tera-printer.
Execução real no Windows e publicação remota continuam pendentes.

### Falha em Validate release prerequisites no Windows

O pré-requisito usa `scripts/check-release.cjs` em vez de comparar strings pelo
Bash. A saída do GitHub CLI é normalizada com trim(), incluindo CRLF do Windows.
Variáveis ausentes, HTTP 401/403/404, repositório não público e divergência de tag
agora produzem mensagens específicas, sem imprimir tokens ou stderr bruto da API.
Os campos CSC_*/APPLE_* vazios são esperados no job Windows sem assinatura.

A tag precisa corresponder à versão do package.json no commit executado. Para
0.1.2, a tag esperada é printer-v0.1.2. Se o commit for diferente, publique uma nova
tag com a versão correta; não sobrescreva releases já distribuídas.

O trecho com a listagem do shell não identifica sozinho qual comando falhou.
Foi confirmado por consulta pública que Ilannildo/teraprinter é público; isso não
valida o token privado usado no runner. Testes da correção: 12 testes de distribuição
aprovados, CLI completa simulada com CRLF e tag incompatível aprovada, YAML válido,
lint concluído com os 5 avisos CSS existentes. Nenhuma release foi publicada nem
execução remota do Actions foi disparada durante essa correção.

### Typecheck no runner limpo: fiscal-danfe/thermal e /demo

Os exports de @teragestor/fiscal-danfe apontam para dist/thermal-layout e
 dist/thermal-demo (JavaScript e declarações TypeScript). Instalar o workspace não
compila essa biblioteca automaticamente. Arquivos dist existentes na máquina local
podem esconder a falta do build no Actions.

`build:dependencies` usa `turbo run build --filter="tera-printer^..."` para compilar
as dependências do Printer em ordem, incluindo fiscal-danfe e print-agent-contract.
Os hooks pretypecheck/prebuild executam essa preparação; o workflow de release
prepara as dependências antes de tipos/testes e o workflow fiscal também inclui o
grafo do Printer. Não foi adicionado alias para contornar os exports compilados.

`node --test scripts/deployment/test-printer-workspace-build.cjs` reproduz TS2307
em um workspace temporário sem dist, reutilizando dependências externas instaladas,
e comprova que o typecheck oficial compila as bibliotecas e passa. O teste também
verifica a presença de .js/.d.ts de thermal/demo e do runtime do contrato. Ele faz
parte do workflow de release. Validações locais: regressão, typecheck, build,
testes de impressão e smoke aprovados. Nova execução no runner Windows ainda
precisa ser feita após enviar o commit corrigido.

### esbuild não encontrado no Actions

Os scripts test:printing, test:updates e smoke:build invocam o CLI esbuild. Ele deve
ser uma devDependency direta de tera-printer: dependências transitivas ou binários
remanescentes de uma instalação local não garantem o executável em uma instalação
filtrada do pnpm. Foi declarado esbuild 0.28.2, versão já presente no lockfile, sem
instalação global, npx ou alteração do runtime de impressão.

Validação da correção: checkout temporário sem node_modules, instalação
`pnpm install --frozen-lockfile --offline --filter=tera-printer...`, seguida de
esbuild --version (0.28.2), build das dependências, 6 testes de impressão, 7 de
update, 12 de distribuição, smoke, typecheck e build: aprovados. Lint concluiu com
os 5 avisos CSS existentes. O ensaio foi executado em macOS; a execução no runner
Windows deve ser confirmada após enviar package.json e pnpm-lock.yaml atualizados.
