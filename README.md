# Painel de Editais e Licitações

## Como publicar no GitHub Pages

1. Suba **toda** esta pasta (`index.html` + `assets/`) para o repositório, na raiz (ou em `/docs`, se preferir).
2. Em **Settings → Pages**, aponte para a branch/pasta onde estes arquivos ficaram.
3. O endereço do site (`https://seu-usuario.github.io/seu-repo/`) nunca muda, mesmo quando você atualizar qualquer arquivo depois.

Não há build, não há dependências para instalar — é um site estático puro.

## Por que os dados não se perdem mais

Antes, os dados viviam apenas na memória do navegador (ou dependiam de uma
API disponível só dentro do ambiente de artefatos da Claude). Isso não
funciona em um site publicado normalmente: recarregar a página, fechar o
navegador, ou publicar uma nova versão do `index.html` apagava tudo.

Agora a aplicação está organizada em três camadas **completamente separadas**:

```
site/
├── index.html                          ← estrutura da página (interface)
├── assets/
│   ├── css/styles.css                  ← aparência
│   └── js/
│       ├── config.js                   ← configuração (cores, chaves, backend ativo)
│       ├── storage/
│       │   ├── storage-adapter.js      ← contrato comum a qualquer backend
│       │   ├── local-storage-adapter.js← backend ATIVO hoje (localStorage)
│       │   ├── supabase-adapter.js     ← backend pronto, desativado por padrão
│       │   └── index.js                ← escolhe o backend e expõe `DataStore`
│       ├── data/
│       │   └── google-agenda-seed.js   ← DADOS (lista de editais do Google Agenda)
│       ├── integrations/
│       │   └── google-agenda-sync.js   ← lógica que aplica os dados acima
│       ├── app.js                      ← lógica/interface do quadro (sem dados embutidos)
│       └── main.js                     ← inicialização
```

**Regra de ouro:** `app.js` nunca fala diretamente com `localStorage` nem
com nenhum banco. Ele só chama `DataStore.get(...)` e `DataStore.set(...)`.
Isso é o que garante duas coisas ao mesmo tempo:

- **Os dados persistem de verdade**: eles ficam salvos no `localStorage` do
  navegador, que sobrevive a F5, fechar o navegador e abrir de novo.
- **Você pode editar o código livremente**: trocar `index.html`, `app.js`
  ou o CSS e publicar de novo não apaga nada, porque o código e os dados
  vivem em lugares diferentes (arquivos vs. armazenamento do navegador).

### Limitação importante do `localStorage`

`localStorage` é por navegador/dispositivo. Se alguém abrir o site em outro
computador, ou limpar os dados do site no navegador, os dados salvos ali não
aparecem. Para um quadro realmente centralizado (visível igual em qualquer
dispositivo), o próximo passo é migrar para o Supabase — a estrutura já foi
criada pensando nisso, veja abaixo.

### Backup manual (funciona hoje, sem depender de nada externo)

Na barra lateral, os botões **"exportar backup (.json)"** e **"importar
backup (.json)"** deixam você baixar todos os dados a qualquer momento e
restaurá-los depois — inclusive como forma de mover os dados de um
computador para outro enquanto o Supabase não estiver ativo.

## Migrando para Supabase

Quando quiser dados centralizados (o mesmo quadro em qualquer dispositivo):

1. Crie um projeto gratuito em [supabase.com](https://supabase.com).
2. No **SQL Editor** do projeto, rode o SQL que já está comentado no final
   de `assets/js/storage/supabase-adapter.js` (cria a tabela `app_storage`
   e libera acesso a ela).
3. Em `assets/js/config.js`, preencha:
   ```js
   const SUPABASE_CONFIG = {
     url: 'https://SEU-PROJETO.supabase.co',
     anonKey: 'SUA-CHAVE-ANON-PUBLICA'
   };
   ```
   e troque `STORAGE_BACKEND` para `'supabase'`.
4. No `index.html`, descomente a linha do script do `supabase-js` (já está
   marcada com um comentário `<!-- Para ativar o Supabase... -->`).
5. Publique. **Nenhum outro arquivo precisa mudar** — `app.js`,
   `google-agenda-sync.js` etc. continuam chamando `DataStore` normalmente.
6. Para não perder o que já estava salvo no `localStorage`: exporte o
   backup (botão na barra lateral) **antes** de trocar o backend, depois
   troque para Supabase e importe esse mesmo arquivo — o botão
   "importar backup" funciona com qualquer backend ativo.

## Integrando o Google Agenda de verdade

Hoje, `assets/js/data/google-agenda-seed.js` contém uma lista de editais
colada manualmente (do jeito que já vinha sendo feito). A lógica que
transforma essa lista em tarefas do quadro está isolada em
`assets/js/integrations/google-agenda-sync.js`, na função
`syncGoogleAgendaEvents()`.

O próximo passo natural é substituir o conteúdo de `google-agenda-seed.js`
por uma leitura real da API do Google Calendar (via OAuth), montando o
mesmo formato de lista que já existe hoje. Como o restante do app só
conhece `syncGoogleAgendaEvents()` — nunca a lista bruta — essa troca não
exige mexer em mais nada.

## Arquivos alterados nesta reestruturação

| Arquivo | O que mudou e por quê |
|---|---|
| `index.html` | Reduzido a só a estrutura da página. CSS e JS foram extraídos para arquivos próprios; nenhum dado embutido. Textos da barra lateral atualizados para refletir armazenamento local; adicionados botões de exportar/importar backup. |
| `assets/css/styles.css` | Todo o CSS que antes estava dentro de `<style>` no `index.html`, sem nenhuma mudança de conteúdo. |
| `assets/js/config.js` | **Novo.** Constantes de configuração (cores, listas fixas, chaves de armazenamento, qual backend está ativo). |
| `assets/js/storage/storage-adapter.js` | **Novo.** Define o contrato que qualquer backend de armazenamento precisa seguir. |
| `assets/js/storage/local-storage-adapter.js` | **Novo.** Implementação real usando `localStorage` — é o que resolve a persistência hoje. |
| `assets/js/storage/supabase-adapter.js` | **Novo.** Implementação pronta para Supabase, desativada até você preencher as credenciais. |
| `assets/js/storage/index.js` | **Novo.** Escolhe o backend configurado e expõe `DataStore`, o único ponto de acesso a dados usado pelo resto do app. |
| `assets/js/data/google-agenda-seed.js` | Antes era um array `GOOGLE_EVENTS_TO_IMPORT` embutido no meio do `<script>` principal. Agora é um arquivo de dados isolado, sem lógica junto. |
| `assets/js/integrations/google-agenda-sync.js` | Antes era a função `importGoogleEvents()` misturada ao resto do código. Agora é um módulo próprio (`syncGoogleAgendaEvents()`), pronto para um dia ler direto da API do Google Calendar. |
| `assets/js/app.js` | O restante da lógica original (renderização do quadro, calendário, relatórios, modais etc.), com todas as chamadas a `window.storage` trocadas por `DataStore` — e com as funções de exportar/importar backup adicionadas ao final. |
| `assets/js/main.js` | **Novo.** Reúne a inicialização que antes ficava solta no fim do `<script>`: carregar dados, sincronizar com o Google Agenda, mostrar a interface, ligar os botões de backup. |
