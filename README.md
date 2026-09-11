# cury-assets

Biblioteca central de imagens dos empreendimentos **Cury Construtora RJ** — fotos e plantas,
espelhadas da CDN `cury.net` e servidas por **jsDelivr** (CDN grátis).

**Por quê:** os sites de corretores puxavam as imagens direto de `cury.net`. Se a Cury mudar
os hashes, migrar de storage ou bloquear hotlink, **todos os sites quebram de uma vez**. Aqui
está o backup independente — e o hash original é preservado, então migrar qualquer site é um
único find/replace.

> ⚠️ Já há links quebrados na origem: 2 imagens referenciadas nos sites já dão **404** em
> `cury.net`. O backup foi feito a tempo (2026-08-17).

## Estrutura

```
gallery/<hash>.jpg          # fotos oficiais Cury (hash = o mesmo de cury.net)
plants/<hash>.jpg           # plantas oficiais
proprias/<slug>/NN.jpg      # imagens que NÃO são da Cury (ex.: Residencial Erê)
manifest.json               # slug -> { name, gallery[], plants[], own[] } com URLs jsDelivr
```

Imagens comprimidas (resize 1600px, JPEG q82 progressivo, sem metadados) — **95 MB no total,
média ~160 KB, nenhuma acima de 500 KB** (medido em 11/09/2026, 604 arquivos). Até 11/09 o
espelho tinha os bytes ORIGINAIS da Cury (199 MB, até 2,1 MB por foto) — a compressão foi feita
com ImageMagick, mantendo só o resultado quando ele é menor que o original.

## Como usar num site

URL base (jsDelivr):
```
https://cdn.jsdelivr.net/gh/fifaosales/cury-assets@main/<caminho>
```
Ex.: `https://cdn.jsdelivr.net/gh/fifaosales/cury-assets@main/gallery/6a04c9be71828.jpg`

Sites novos: consumir o `manifest.json` (por slug) ou apontar direto pras URLs acima.

## 🔴 Migrar um site de cury.net → jsDelivr (find/replace único)

Como o hash é preservado, trocar TODA a origem das imagens de um site é um regex só.
Rode na raiz do repositório do site (ajuste a extensão dos arquivos):

```bash
# preview
grep -rlE "cury\.net/storage/images(_webp)?/products/(gallery|plants)/[0-9a-f]+\.jpeg" src/

# aplicar (Node)
node -e '
const fs=require("fs"),cp=require("child_process");
const files=cp.execSync(`grep -rlE "cury\\.net/storage/images" src/`).toString().split("\n").filter(Boolean);
const re=/https:\/\/cury\.net\/storage\/images(?:_webp)?\/products\/(gallery|plants)\/([0-9a-f]+)\.jpeg(?:\.webp)?/g;
const base="https://cdn.jsdelivr.net/gh/fifaosales/cury-assets@main";
for(const f of files){const s=fs.readFileSync(f,"utf8");const n=s.replace(re,(_,t,h)=>`${base}/${t}/${h}.jpg`);if(n!==s){fs.writeFileSync(f,n);console.log("migrado",f)}}
'
```

## Estratégia de rollout

- **Backup completo** (este repo) = o seguro. Feito uma vez, mantido atualizado.
- **Sites novos** já nascem apontando pro jsDelivr.
- **Sites existentes** migram quando forem mexidos — ou todos de uma vez (regex acima) se a
  Cury bloquear a origem.

## 🔒 Segurança — o que pode e o que NÃO pode entrar aqui

Repo **público** (requisito do jsDelivr). Uma "key" não protegeria nada: as imagens
aparecem nos sites e já são abertas em `cury.net`. A proteção é sobre **o que entra**:

- ✅ **PODE:** imagens de divulgação (fotos, plantas) e a infra (`manifest.json`, `README`).
- ❌ **NUNCA:** dados de lead, `.env`, credenciais, tabelas de preço internas, contratos,
  documentos, planilhas, dumps de banco.

Barreira técnica: o `.gitignore` é **allowlist** — bloqueia tudo e libera só imagem +
`manifest.json` + `README`. Se você precisar de `git add -f` pra versionar algo aqui,
**pare** e confirme que aquilo pode ser público.

## Atualizar (novos empreendimentos / novas fotos)

Clones: `~/projetos/cury-assets` (WSL, fonte) e `C:\github\cury-assets` (espelho Windows).
**Toda imagem entra comprimida** — foi assim que o repo caiu de 199 MB para 95 MB em 11/09/2026, e
o que entrar pesado a partir daqui pesa em todos os sites de corretores ao mesmo tempo.

> **No WSL** (`cd ~/projetos/cury-assets`; ImageMagick 6 → o binário é `convert`):
> ```bash
> # 1. baixar da Cury, preservando o hash (gallery ou plants)
> h=6a8628c0a493b; t=gallery
> curl -sL -A "Mozilla/5.0" "https://cury.net/storage/images/products/$t/$h.jpeg" -o "$t/$h.jpg"
> # 2. comprimir — só substitui se ficar menor
> convert "$t/$h.jpg" -auto-orient -strip -resize '1600x1600>' -sampling-factor 4:2:0 -interlace JPEG -quality 82 /tmp/c.jpg
> [ "$(stat -c%s /tmp/c.jpg)" -lt "$(stat -c%s "$t/$h.jpg")" ] && mv /tmp/c.jpg "$t/$h.jpg"
> # 3. gate: nada acima de 500 KB
> find gallery plants proprias -name '*.jpg' -size +500k
> # 4. manifest.json (slug -> gallery/plants/own) e push
> git add -A && git commit -m "mirror: <empreendimento>" && git push
> ```

**Se um arquivo que JÁ existia mudou de bytes**, o jsDelivr pode servir a versão velha por até 12h:
`curl -s "https://purge.jsdelivr.net/gh/fifaosales/cury-assets@main/$t/$h.jpg"`. Conferir com um
`HEAD` — o `content-length` tem que bater com `stat -c%s` do arquivo local. Arquivo novo não precisa.

**Migrar um site que ainda puxa de `cury.net`:** o find/replace da seção acima. Antes, confira a
cobertura (todo hash do site existe aqui), senão vira 404 na cara do cliente:

```bash
# na raiz do site
for p in $(grep -ohE "products/(gallery|plants)/[0-9a-f]+\.jpeg" -r src/ | sed 's#products/##; s#\.jpeg##' | sort -u); do
  [ -f ~/projetos/cury-assets/$p.jpg ] || echo "FALTA $p"
done   # tem que sair vazio
```
