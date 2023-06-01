# 섹션 27, 파일 업로드
## 요약
  - `<input type="file">`을 사용하여 `파일선택기` 추가
    - `accept="image/jpg,image/png"` 업로드 파일 제한
  - `<form enctype="multipart/form-data">` 인코딩 타입 설정
  - `multer` 타사 패키지 설치 및 라우트 설정
    - `npm install --save multer` 설치
    - `const multer = require('multer')` 설정 및 구성(path,filename)
    - `const storageConfig = multer.diskStorage({})` 
      - `destination: function(req,file,cb) {cb(null, 'images')}`
      - `filename: function(req, file, cb) {cb(null, Date.now() + '-' + file.originalname)}`
  - 데이터베이스 경로 저장 라우트 설정
    - `router.post('/profiles', upload.single('image'), async function(req, res) {})`
      - `upload.single('name')`은 `<input type="file">`의 `name 속성`과 일치 해야 함.
    - `const uploadedImageFile = req.file;`
    - `const userData = req.body;`
    - `await db.getDb().collection('users').insertOne({})`
      - `name: userData.username`
      - `imagePath: uploadedImageFile.path`
  - 템플릿 설정
    - `router.get('/', async function(req, res){})`
      - `const users = await db.getDb().collection('users').find().toArray();`
      - `res.render('profiles', { users: users })`
    - `<img src="<%= user.imagePath %>" alt="The image of the user.">`
    - `<h2><%= user.name %></h2>`
  - 미들웨어 설정
    - `app.use('/images', express.static('images'));`

## 1) 파일선택기 추가 (인풋 태그 (파일 선택과 추가))

- 파일 선택 버튼 생성 : `<input type="file" id="image" name="image" required>`
- `required` 설정
- 업로드 파일 허용
  - `accept=".jpg,.png,.jpeg"` 허용 확장자
  - `accept="image/jpg,image/png` jpg 이미지 파일

## 2) 폼 태그 속성(인코딩 설정) 추가 `enctype`

- 브라우저가 서버로 업로드 된 파일을 보낼 때 인코딩 방식 설정
- `<form action="/profiles" method="POST" enctype="application/x-www-form-urlencoded">` Defualt
- `enctype="multipart/form-data"` 하나 이상의 파일선택기일 때

## 3) "multer" 패키지로 들어오는 파일 업로드 구문 분석 추가

- `app.user(express.urlencoded({ extended: false}));`
  - expressJS에 내장된 urlencoded 미들웨어로 들어오는 요청에 대한 구문 분석과 데이터 추출
  - `파일`을 포함하는 `multipart/form-data`를 처리할 수 없음
- 타사 패키지 사용 `multer` for `multipart/form-data`
  - `npm` 또는 `github`에서 찾을 수 있음
  - `npm install --save multer` 설치
  - `모든 요청`에 적용하지 않고 `해당 라우트`에서만 사용

```JavaScript
// users.js
const multer = require('multer');
const upload = multer({});

// 라우트의 추가 매개변수로 미들웨어 함수목록 추가 가능
// 왼쪽에서 오른쪽으로 실행
// upload.single('name')
router.post('/profiles', upload.single('image') , function(req,res) {
    // upload된 파일에 접근
    const uploadedImageFile = req.file;
    // 폼 데이터 접근
    const userData = req.body;
})
```

## 4) 업로드 된 파일 저장 (확장자 설정 안됨)

- 데이터베이스는 파일 스토리지에 적합하지 않음
- 파일 시스템(Hard Drvie 등)에 저장해야 함
- 데이터베이스에는 `파일경로`가 저장되어야 함

```JavaScript
// users.js
const multer = require('multer');
// 파일이 저장될 경로
const upload = multer({dest : 'images' }); // dest : destination

router.post('/profiles', upload.single('image') , function(req,res) {
    // 저장된 이미지 파일 경로
    const uploadedImageFile = req.file;
    const userData = req.body;

    console.log(uploadedImageFile);
    console.log(userData);

    res.redirect('/');
})
```

## 5) 저장된 파일의 확장자 설정 후 파일 저장

- `multer` 구성
- `Date.now()` : 밀리초 단위의 숫자로 현재날짜 반환

```JavaScript
// users.js
const multer = require('multer');

const storageConfig = multer.diskStorage({
    destination: function (req, file, cb) {
        cb(null, 'images');
    },
    filename: funtion(req, file, cb) {
        cb(null, Date.now() + '-' + file.originalname);
    }
})
const upload = multer({storage : storageConfig });

router.post('/profiles', upload.single('image') , function(req,res) {
    const uploadedImageFile = req.file;
    const userData = req.body;

    console.log(uploadedImageFile);
    console.log(userData);

    res.redirect('/');
})
```

## 6) 데이터베이스에 저장된 파일 경로, 필요한 정보 저장

```JavaScript
// users.js
const db = require('../data/database');

router.post('/profiles', upload.single('image') , async function(req,res) {
    const uploadedImageFile = req.file;
    const userData = req.body;

    await db.getDb().collection('users').insertOne({
      name: userData.username,
      imagePath: uploadedImageFile.path
    });

    res.redirect('/');
})
```

## 7) 웹사이트 방문자에게 업로드된 파일 제공

- 데이터베이스 조회 후 조회 데이터 전달

```JavaScript
router.get('/', async function(req,res) {
  const users = await db.getDb().collection('users').find().toArray();
  res.render('profiles', {users : users});
})
```

- 클라이언트는 서버 측 폴더 경로로 접근을 할 수 없음.
- 정적 미들웨어 추가 설정
  - `app.use('/images', express.static('images'));`
  - `/images` 로 시작하는 요청 시에만 작동하는 미들웨어
  - `/images/filename` 으로 작동하게 됨.
- 만약, `/images` 필터링이 없다면 `/images/filename`이 아니라 `/filename` 경로가 됨

```html
<!-- profiles.ejs -->
<section id="user-profiles">
  <ul>
    <% for (const user of users) { %>
    <li class="user-item">
      <article>
        <img src="<%= user.imagePath %>" alt="The image of the user." />
        <h2><%= user.name %></h2>
      </article>
    </li>
    <% } %>
  </ul>
</section>
```

## 8) 이미지 미리보기 기능 추가

- 자바스크립트로 새 페이지를 로드하지 않고 작동
- `public/scripts/file-preview.js` 생성
- `new-user.ejs`에 스크립트 링크 추가
  - `<script src="/scripts/file-preview.js" defer></script>`
  - `defer` 페이지의 나머지 부분이 구문분석된 후 로드하도록 설정
- 파일 입력(`<input type="file">`)에 `리스너` 추가
  - `new-user.ejs`
    - `image-preview` 설정

```html
<!-- new-user.ejs -->
<div class="form-control">
  <img id="image-preview" src="" alt="Your picked image." />
  ...
</div>
```

- `file-preview.js`
  - `<input>`,`<img>` 요소의 참조 생성
  - `이벤트리스너 'chnage'` 추가
    - 선택한 값이 변경될 때마다 작동
    - 리스너 추가 매개변수 `function` 추가
      - function의 `()`를 빼서 이벤트발생 때만 동작
      - `.files` 속성으로 업로드 된 파일에 접근
      - `if문` 추가
        - 파일이 없을 경우 `return`, `display:none`
  - `URL.createObjectURL` 
    - 블롭 객체를 가리키는 URL 생성 
    - 브라우저 메모리 공간 외부에서는 사용 불가능.
```JavaScript
// file-preview.js
const filePickerElement = documnet.getElementById('image');
const imagePreviewElement = document.getElementById('image-preview');

function showPreview() {
  const files = filePickerElement.files;
  if (!files || files.length === 0) {
    imagePreviewElement.style.display= 'none';
    return;
  }
  
  const pickedFile = files[0];

  imagePreviewElement.src = URL.createObjectURL(pickedFile)
  imagePreviewElement.style.display= 'block';

}

filPickerElement.addEvenetListener('change', showPreview);
```
  - `CSS` 설정
    - `#image-preview`
      - width: 2.5rem
      - height: 2.5rem
      - object-fit: cover;
      - display: none;

```CSS
#image-preview {
  width: 2.5rem
  height: 2.5rem;
  object-fit: cover;
  display:none;
}
```