# JWT 認証の仕組み

## プロジェクトフォルダを作成
```bash
mkdir express && cd express
touch server.js
npm init -y
npm install express nodemon
```
## 認証サーバ 環境を設定
```bash
npm install express nodemon
vi package.json
```
`npm start` で `nodemon` 経由で `server.js` が起動出来るように `package.json` の `scripts` に `"start": "nodemon server.js"` を追記する
```json
{
  "name": "express",
  "version": "1.0.0",
  "description": "",
  "main": "server.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "nodemon server.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.1",
    "nodemon": "^2.0.19"
  }
}
```
とりあえず、起動しておきます。
nodemon はファイル保存の度再起動を自動で実行してくれるモジュールになります。
```bash
npm start
```
## 認証ユーザーの新規登録の流れ
1. ユーザーのアカウントIDと、パスワードを入力
2. バリデーションチェック
3. すでにユーザーが存在しているか確認
4. パスワードの暗号化
5. DBに保存
6. JWT発行してクライアントに渡す

### 1. ユーザーのアカウントIDと、パスワードを入力
クライアントからのアカウントID
IDとパスワードの入力は、ここでは `postman`と言うアプリケーションで
入力内容を送信します。
サーバー側は以下の様に記述します。
```bash
mkdir routes
vi routes/auth.js
```
```javascript
const router = require('express').Router();

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up', (req, res) => {
	const acount = req.body.acount;
	const password = req.body.password;
	console.log(acount, password);
});

module.exports = router;
```
サーバールーティング処理を追記する。
```bash
vi server.js
```
```javascript
const express = require('express');
const app = express();
const SERVER_PORT = process.env.SERVER_PORT || 5000;
const auth = require('./routes/auth');

app.use(express.json());
app.use('/auth', auth);

app.get('/', (req, res) => {
	res.send('Hello Express');
});

app.listen(SERVER_PORT, () => {
	console.log('server start')
	console.log(`http://localhost:${SERVER_PORT}`)
})
```
`postman` で http://localhost:5000/auth/sign-up にアクセスすると
コンソールに acount 入力情報と password 入力情報が表示されます。

### 2. バリデーションチェック
バリデーションのモジュールをインストールする。
```bash
npm install express-validator
```
バリデーション処理を記述する。
```bash
vi routes/auth.js
```
```javascript
const router = require('express').Router();
const { body, validationResult } = require('express-validator');

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up',
	// 2. バリデーションチェック
	body('acount').isLength({ min: 6 }).isAlpha(),
	body('password').isLength({ min: 6 }),
	(req, res) => {
		const acount = req.body.acount;
		const password = req.body.password;

		const errors = validationResult(req);
		if (!errors.isEmpty()) {
			return res.status(400).json({ errors: errors.array() });
		}
		console.log(acount, password);
	});

module.exports = router;
```
### 3. すでにユーザーが存在しているか確認
ユーザー認証情報を管理するために今回は、DBとして`sqlite`をインストールします。
```bash
npm install sqlite3
```
DBの操作を容易にするモジュールとして`Prisma`をインストールします。
Prisma とは ORM になります。
```bash
npm install prisma
npx prisma init
```
Prisma を利用するには .env ファイルに必要情報を記述していきます。
```bash
vi .env
```
```env
DATABASE_URL="file:./data.db"
```
次に行うのは、Prisma のデータベース利用に関する設定情報です。
`prisma`フォルダの中に`shema.prisma`と言うファイルを開いて以下の様に記述します。
```bash
vi prisma/shema.prisma
```
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}
```
次に行うのはユーザー情報を保存するテーブルは「モデル」として定義します。
上記の`shema.prisma`ファイルに下記の様に追記します。
```prisma
model User {
  id Int @id @default(autoincrement())
  acount String @unique
  password String
  updatedAt DateTime @updatedAt
}
```
モデル情報を記述できましたら、DBにモデルに対応するテーブル作成を下記のコマンドで実行します。
```bash
npx prisma migrate dev --name initial
```
これで、DBにテーブルが用意されましたので、ダミーデータを登録していきますが、Prisma には便利なUIがあって下記のコマンドでWEBUIにてテーブルにデータが登録できますのでダミーデータを登録しておきます。
```bash
npx prisma studio
```
登録が完了したら、
ユーザー存在の処理を記述する。
```bash
vi routes/auth.js
```
```javascript
const router = require('express').Router();
const { body, validationResult } = require('express-validator');
const ps = require('@prisma/client');
const prisma = new ps.PrismaClient();

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up',
	// 2. バリデーションチェック
	body('acount').isLength({ min: 6 }).isAlpha(),
	body('password').isLength({ min: 6 }),
	(req, res) => {
		const acount = req.body.acount;
		const password = req.body.password;

		const errors = validationResult(req);
		if (!errors.isEmpty()) {
			return res.status(400).json({ errors: errors.array() });
		}
		// 3. すでにユーザーが存在しているか確認
		const user = await prisma.user.findFirst({
			where: {
				acount: acount
			}
		});
		if (user) {
			return res.status(400).json({
				message: 'すでに、ユーザーは登録されています。'
			})
		}
		console.log(acount, password);
	});

module.exports = router;
```

### 4. パスワードの暗号化
パスワードの暗号化のモジュールをインストールする。
```bash
npm install bcrypt
```
ここでは、暗号化とは、パスワード文字列をハッシュ化する事を言います。
パスワードの暗号化の処理を記述する。
```bash
vi routes/auth.js
```
```javascript
const router = require('express').Router();
const { body, validationResult } = require('express-validator');
const ps = require('@prisma/client');
const prisma = new ps.PrismaClient();
const bcrypt = require('bcrypt');

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up',
	// 2. バリデーションチェック
	body('acount').isLength({ min: 6 }).isAlpha(),
	body('password').isLength({ min: 6 }),
	async (req, res) => {
		const acount = req.body.acount;
		const password = req.body.password;

		const errors = validationResult(req);
		if (!errors.isEmpty()) {
			return res.status(400).json({ errors: errors.array() });
		}
		// 3. すでにユーザーが存在しているか確認
		const user = await prisma.user.findFirst({
			where: {
				acount: acount
			}
		});
		if (user) {
			return res.status(400).json({
				message: 'すでに、ユーザーは登録されています。'
			})
		}
		// 4. パスワードの暗号化
		let hashedPassword = await bcrypt.hash(password, 10)
		console.log(hashedPassword)
	});

module.exports = router;
```

### 5. DBに保存
ユーザー登録の処理を記述する。
```bash
vi routes/auth.js
```
```javascript
const router = require('express').Router();
const { body, validationResult } = require('express-validator');
const ps = require('@prisma/client');
const prisma = new ps.PrismaClient();
const bcrypt = require('bcrypt');

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up',
	// 2. バリデーションチェック
	body('acount').isLength({ min: 6 }).isAlpha(),
	body('password').isLength({ min: 6 }),
	async (req, res) => {
		const acount = req.body.acount;
		const password = req.body.password;

		const errors = validationResult(req);
		if (!errors.isEmpty()) {
			return res.status(400).json({ errors: errors.array() });
		}
		// 3. すでにユーザーが存在しているか確認
		const user = await prisma.user.findFirst({
			where: {
				acount: acount
			}
		});
		if (user) {
			return res.status(400).json({
				message: 'すでに、ユーザーは登録されています。'
			})
		}
		// 4. パスワードの暗号化
		let hashedPassword = await bcrypt.hash(password, 10)
		//console.log(hashedPassword)
		// 5. DBに保存
		await prisma.user.create({
			data: {
				acount: acount,
				password: hashedPassword
			}
		})
	});
module.exports = router;
```

### 6. JWT発行してクライアントに渡す
JWT とは JSON Web Token の略で 認証トークンになります。

JWTのモジュールをインストールする。
```bash
npm install jsonwebtoken
```
環境変数`SECRET_KEY`に暗号化するにあたってのシークレットキーを記述します。
```bash
vi .env
```
```evn
DATABASE_URL="file:./data.db"
SECRET_KEY="secret_key"
```
JWTの処理を記述する。
```bash
vi routes/auth.js
```
```javascript
const router = require('express').Router();
const { body, validationResult } = require('express-validator');
const ps = require('@prisma/client');
const prisma = new ps.PrismaClient();
const bcrypt = require('bcrypt');
const JWT = require('jsonwebtoken');
const SECRET_KEY = process.env.SECRET_KEY || "secret";

router.get('/', (req, res) => {
	res.send('Hello Auth')
});
// ユーザー新規登録用のAPI
router.post('/sign-up',
	// 2. バリデーションチェック
	body('acount').isLength({ min: 6 }).isAlpha(),
	body('password').isLength({ min: 6 }),
	async (req, res) => {
		const acount = req.body.acount;
		const password = req.body.password;

		const errors = validationResult(req);
		if (!errors.isEmpty()) {
			return res.status(400).json({ errors: errors.array() });
		}
		// 3. すでにユーザーが存在しているか確認
		const user = await prisma.user.findFirst({
			where: {
				acount: acount
			}
		});
		if (user) {
			return res.status(400).json({
				message: 'すでに、ユーザーは登録されています。'
			})
		}
		// 4. パスワードの暗号化
		let hashedPassword = await bcrypt.hash(password, 10)
		//console.log(hashedPassword)
		// 5. DBに保存
		await prisma.user.create({
			data: {
				acount: acount,
				password: hashedPassword
			}
		})
		// 6. JWT発行してクライアントに渡す
		const token = await JWT.sign(
			{
				acount
			},
			SECRET_KEY,
			{
				expiresIn: "24h"
			}
		)
		return res.json({
			token: token
		})
	});

// ユーザー一覧を確認するAPI
router.get('/users', async (req, res) => {
	const users = await prisma.user.findMany();
	return res.json(users);
})

// ログインAPI
router.post('/login', async (req, res) => {
	const { acount, password } = req.body;

	const user = await prisma.user.findFirst({
		where: {
			acount: acount
		}
	});
	if (!user) {
		return res.status(400).json([
			{
				message: "ユーザーが存在しません"
			}
		])
	}
	// パスワードの復元、照合
	const isMatch = await bcrypt.compare(password, user.password)
	if (!isMatch) {
		return res.status(400).json([
			{
				message: "パスワードが違います"
			}
		])
	}
	const token = await JWT.sign(
		{
			acount
		},
		SECRET_KEY,
		{
			expiresIn: "24h"
		}
	)
	return res.json({
		token: token
	})
})

module.exports = router;
```