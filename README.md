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

Imagens comprimidas (resize 1600px, mozjpeg q82) — ~250 KB cada, 77 MB no total.

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

## Atualizar (novos empreendimentos / novas fotos)

Re-rodar o script de download (em `curyconstrutoras/scripts` ou no scratchpad) apontando pras
novas URLs `cury.net`, depois `git add . && git commit && git push`. O jsDelivr atualiza em
até ~12h (ou force purge via `purge.jsdelivr.net`).
