# Segurança e configuração — Vértice 3D

Este arquivo reúne os passos que precisam ser feitos **no console** (não dá para
fazer pelo código). São 3 blocos independentes: **A) Login do admin**,
**B) Regras do banco** e **C) Avisos de novo lead**.

---

## A) Login do admin (Firebase Authentication)

A senha antiga (`vertice2025`) ficava escrita no `index.html`. Como o site é
público, qualquer pessoa que abrisse "ver código-fonte" conseguia ler a senha,
entrar no painel e ver os dados dos clientes. Isso foi removido — agora o acesso
usa Firebase Authentication.

**Passos:**

1. Abra o [Firebase Console](https://console.firebase.google.com/) → projeto
   **vertice3d-be624**.
2. Menu **Authentication** → aba **Sign-in method** → ative **Email/Senha**.
3. Aba **Users** → **Add user** → informe o e-mail e uma senha forte do
   responsável. (Essa senha **não** vai para o código — fica só no Firebase.)
4. Ainda em **Users**, copie o **User UID** que aparece na linha do usuário criado.

Pronto. A partir daí o painel (⚙) pede e-mail + senha e valida no Firebase.

> ### ✅ Status desta instalação
> Já concluído em 20/07/2026:
> - Conta admin: **vertice.renders3d@gmail.com**
> - UID: `Nq9Yg2d1Q4UNGBpJMTfK8odBzFk2`
>
> Os blocos de regras do passo B abaixo **já estão com esse UID preenchido** —
> é só copiar e colar, sem editar nada.

> **Recomendado:** em **Authentication → Settings → User actions**, desmarque
> **"Enable create (sign-up)"**. Isso impede que estranhos criem conta sozinhos.
> Mesmo assim, as regras do passo B já travam no UID, que é a proteção real.

---

## B) Regras do banco (Firestore + Storage)

Sem isto, o passo A não protege nada: os dados continuariam legíveis por qualquer
um direto pela API, mesmo sem passar pelo painel.

### Por que travar no UID e não só em "está logado"

Com Email/Senha ativo, qualquer pessoa pode criar uma conta usando a chave
pública do site e ficaria "autenticada". Por isso a regra **não** pode ser só
`request.auth != null` — ela precisa exigir **o UID do admin**.

> **Duas formas de aplicar** — escolha uma:
>
> **(a) Copiar e colar no Console** (mais simples, não precisa instalar nada) —
> é o que está descrito abaixo.
>
> **(b) Pelo Firebase CLI**, se preferir. As regras já estão versionadas no
> repositório (`firestore.rules`, `storage.rules`, `firebase.json`):
> ```bash
> npm install -g firebase-tools
> firebase login
> firebase deploy --only firestore:rules,storage --project vertice3d-be624
> ```

### Firestore

Console → **Firestore Database** → aba **Rules** → cole e **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // >>> COLE AQUI O UID COPIADO NO PASSO A.4 <<<
    function isAdmin() {
      return request.auth != null
          && request.auth.uid == 'Nq9Yg2d1Q4UNGBpJMTfK8odBzFk2';
    }

    // Conteúdo do site: qualquer visitante lê (o site precisa disso), só admin escreve
    match /config/{docId}      { allow read: if true; allow write: if isAdmin(); }
    match /portfolio/{docId}   { allow read: if true; allow write: if isAdmin(); }
    match /depoimentos/{docId} { allow read: if true; allow write: if isAdmin(); }

    // Leads: o visitante SÓ PODE CRIAR (enviar o formulário).
    // Ler / editar / apagar => somente admin. É isto que protege os dados pessoais.
    match /leads/{docId} {
      allow create: if true;
      allow read, update, delete: if isAdmin();
    }

    // Newsletter: mesma lógica
    match /newsletter/{docId} {
      allow create: if true;
      allow read, update, delete: if isAdmin();
    }

    // Qualquer outra coleção: bloqueada
    match /{document=**} { allow read, write: if false; }
  }
}
```

### Storage

Console → **Storage** → aba **Rules** → cole e **Publish**:

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      // Imagens do portfólio e logo precisam ser públicas (aparecem no site)
      allow read: if true;
      // Só o admin envia/apaga arquivo
      allow write: if request.auth != null
                   && request.auth.uid == 'Nq9Yg2d1Q4UNGBpJMTfK8odBzFk2';
    }
  }
}
```

### Como testar se ficou seguro

1. Abra o site numa **janela anônima** (sem login).
2. Console do navegador (F12) e rode:
   ```js
   const {db,collection,getDocs}=window._FB;
   getDocs(collection(db,'leads')).then(s=>console.log('LEU',s.size)).catch(e=>console.log('BLOQUEADO',e.code));
   ```
3. O esperado é **`BLOQUEADO permission-denied`**. Se aparecer `LEU`, as regras
   não foram publicadas corretamente.
4. Depois entre no painel com o admin e confira que a aba **Leads** carrega.

> A `apiKey` que aparece no `index.html` é **normal** ser pública no Firebase —
> ela identifica o projeto, não dá permissão. Quem dá permissão são as regras
> acima. Não precisa escondê-la.

---

## C) Avisos de novo lead (e-mail + WhatsApp)

Antes, o lead só era gravado no banco — não saía aviso nenhum, então só era
visto se alguém abrisse o painel. Agora o site dispara e-mail e WhatsApp.

As chaves ficam no `index.html`, procure por **`WEB3FORMS_KEY`** (tem um
comentário grande explicando). **Enquanto estiverem vazias, o lead continua
sendo salvo e aparece no painel normalmente — só não dispara o aviso.**

### 1. E-mail — Web3Forms (grátis, sem criar conta)

1. Acesse <https://web3forms.com>.
2. Informe o e-mail que deve **receber** os leads e confirme.
3. Chega um **Access Key** nesse e-mail.
4. Cole em `WEB3FORMS_KEY` no `index.html`.

Plano gratuito: 250 e-mails/mês.

### 2. WhatsApp — CallMeBot (grátis, sem criar conta)

1. Salve nos contatos o número **+34 644 51 95 23**.
2. Envie para ele, pelo WhatsApp, a frase exata:
   ```
   I allow callmebot to send me messages
   ```
3. O bot responde com o seu **apikey**.
4. No `index.html`, preencha:
   - `CALLMEBOT_APIKEY` → o apikey recebido;
   - `CALLMEBOT_PHONE` → o número que vai **receber** o aviso, com DDI e só
     dígitos (ex.: `5541999998888`).

### Limitação honesta

O site é estático (GitHub Pages, sem servidor), então **essas duas chaves ficam
visíveis no código-fonte**. Na prática o risco é alguém usá-las para mandar spam
para o seu próprio e-mail/WhatsApp — não dá acesso a dado de cliente. Se
acontecer, gere chaves novas nos dois serviços e substitua.

Para eliminar esse risco de vez seria preciso um servidor (Firebase Cloud
Functions no plano Blaze), onde as chaves ficariam escondidas. Fica como
evolução futura.
