<template>
  <dl class="track-quiz">
    <dt class="title">譜面クイズ</dt>
    <dd class="list">
      <p style="padding:8px; text-align:center; font-size:12px; color:#666;">※ 画像をクリックすると答えに飛びます</p>
      <ul>
        <li class="quiz" v-for="quiz in quizsSorted" key="quiz.id">
          <a :href="quiz.answerUrl" target="_blank">
            <img :src="quiz.imageSrc" :alt="quiz.id" />
          </a>
        </li>
      </ul>
    </dd>
    <dd class="upload">
      <div v-if="!isShowAddQuizForm" @click="showAddQuizForm">
        <img src="img/icon-plus.png" style="width:80px; height:auto;" />
      </div>
      <div v-if="isShowAddQuizForm" style="padding:16px 4% 32px 4%; background-color:#faf0ff;">
        <track-quiz-form v-model="quizAnswerUrl"></track-quiz-form>
        <label class="quiz-image" for="quiz-image-input">＋ 画像を選択 {{quizImageName}}
          <input id="quiz-image-input" type="file" accept="image/*" @change="setQuizImage" style="display:none;">
        </label>
        <img :src="previewImageSrc" alt="プレビューが表示されます" style="display:block; width:30%; padding:4px; font-size:10px; white-space:nowrap;"/>
        <input class="btn-add-quiz" type="button" value="クイズを追加する" @click="addQuiz">
      </div>
    </dd>
  </dl>
</template>

<script>
import trackQuizForm from './trackQuizForm.vue';
// use cdn instead of node module (for reduce build file)
import AWS from 'aws-sdk';  // need for AWS.CognitoIdentityCredentials. (want to import partially)
//import S3 from '../../node_modules/aws-sdk/clients/s3';

export default {
  components: {
    'track-quiz-form': trackQuizForm
  },
  data() {
    return {
      isShowAddQuizForm: false,
      quizsS3: [],
      quizs: [],
      quizImage: null,
      quizAnswerUrl: '',
      previewImageSrc: '',
      s3: null,
      quizJsonFileName: 'quiz.json'
    };
  },
  computed: {
    quizImageName() {
      return this.quizImage ? this.quizImage.name : '';
    },
    quizsSorted() {
      return this.quizs.slice().reverse();
    }
  },
  mounted() {
    // AWS設定
    const REGION = 'ap-northeast-1';
    const IDENTITY_POOL_ID = 'ap-northeast-1:f2418ccb-5acb-4223-ae10-49191f31c2a6';
    const BUCKET = 'sdvx-quiz';
    AWS.config.update({
      region: REGION,
      credentials: new AWS.CognitoIdentityCredentials({IdentityPoolId: IDENTITY_POOL_ID})
    });
    this.s3 = new AWS.S3({params: {Bucket: BUCKET}});
    this.fetchQuizs();
  },
  methods: {
    // クイズ画像設定
    setQuizImage(e) {
      if (!e.target.files[0]) {
        return;
      }
      this.quizImage = e.target.files[0];

      // プレビューを表示する
      const self = this;
      const reader = new FileReader();
      reader.readAsDataURL(this.quizImage);
      reader.onload = function() {
        self.previewImageSrc = reader.result;
      }
    },

    // クイズ追加フォーム表示
    showAddQuizForm() {
      this.isShowAddQuizForm = true;
    },

    // クイズ追加
    addQuiz(e) {
      if (!this.quizImage || !this.quizAnswerUrl) {
        if (!this.quizImage) {
          alert('画像が選択されていません');
        }
        if (!this.quizAnswerUrl) {
          alert('楽曲が選択されていません');
        }
        return;
      }

      // 入力ファイルのデータ取得、S3アップロード、および、クイズリスト更新
      // S3に画像バックアップをアップロード
      const imageFile = this.quizImage;
      const imageFileName = Date.now() + '_' + this.quizImage.name;
      this.s3.putObject({
        Key: imageFileName,
        Body: imageFile,
        ACL: 'public-read'
      }, function(err, data) {
        if (err) { alert('画像保存失敗 : ' + err); }
      });

      // S3に更新版クイズリストをアップロード
      const id = imageFileName;
      const imageSrc = 'https://s3-ap-northeast-1.amazonaws.com/sdvx-quiz/' + imageFileName;
      const answerUrl = this.quizAnswerUrl;
      this.quizsS3.push({
        id,
        imageSrc,
        answerUrl
      });
      const jsonFile = JSON.stringify(this.quizsS3);
      const jsonFileName = this.quizJsonFileName;
      const jsonFileType = 'application/json';
      this.s3.putObject({
        Key: jsonFileName,
        ContentType: jsonFileType,
        Body: jsonFile,
        ACL: 'public-read'
      }, function(err, data){
        if (err) { alert('クイズリスト更新失敗 : ' + err); }
      });

      // クイズリスト表示更新（一時的にbase64で追加し、即時反映）
      const self = this;
      const reader = new FileReader();
      reader.readAsDataURL(this.quizImage);
      reader.onload = function() {
        // クイズリスト更新
        self.quizs.push({
          id,
          imageSrc: reader.result,
          answerUrl
        });

        // 一時データを初期化
        self.quizImage = null;
        self.quizAnswerUrl = '';
        self.previewImageSrc = '';
      }
    },

    // S3からクイズリスト取得
    fetchQuizs() {
      const self = this;
      this.s3.getObject({
        Key: self.quizJsonFileName
      }, function(err, data) {
        if (err) { alert('クイズリスト取得失敗 : ' + err); }
        if (data) {
          const json = JSON.parse(data.Body.toString());
          self.quizsS3 = json;
          self.quizs = json.slice();
        }
      });
    }
  }
};
</script>

<style scoped>
.track-quiz {
  padding: 20px 4% 60px 4%;
  min-height: inherit;
  .title {
    text-align: center;
    padding: 8px 0 16px 0;
    font-size: 24px;
    color: #555;
  }
  .list {
    .quiz {
      display: inline-block;
      vertical-align: top;
      width: 30%;
      padding: 10px 1.5%;
      img {
        width: 100%;
        height: auto;
      }
    }
  }
  .upload {
    .quiz-image {
      display: block;
      margin-top: 8px;
      padding: 0 3%;
      line-height: 40px;
      font-size: 14px;
      color: #888;
      border: 1px solid #bebebe;
      border-radius: 2px;
      box-sizing: border-box;
    }
    .btn-add-quiz {
      padding: 12px 16px;
      margin-top: 32px;
      font-size: 14px;
      color: #555;
      background-color: #ddddff;
      border-radius: 4px;
      border: none;
    }
  }
}
</style>
