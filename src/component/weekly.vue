<template>
  <dl class="weekly">
    <dt class="title">今週の曲 ♫</dt>
    <dd style="padding:8px; text-align:center; font-size:12px; color:#666;">各LEVELの曲選びに迷ったら使ってみてね</dd>
    <dd class="track" v-for="track in tracks">
      <a :href="'http://www.sdvx.in' + track.path + '.htm'" target="_blank">
        <div class="track-image-container">
          <img :src="'http://www.sdvx.in/' + track.ver + '/jacket/'+ track.id + '.png'" alt=""/>
        </div>
        {{track.name}}
      </a>
    </dd>
  </dl>
</template>

<script>
export default{
  mounted() {
    this.fetchWeeklyTracks();
  },
  data() {
    return {
      tracks: []
    };
  },
  methods: {
    // 今週のtracksを取得
    fetchWeeklyTracks() {
      const self = this;
      const url = 'data/weekly.json';
      fetch(url).then(function(res) {
        return res.json();
      }).then(function(json){
        self.tracks = json;
      });
    }
  }
};
</script>

<style scoped>
.weekly {
  padding: 20px 4%;
  min-height: inherit;
  .title {
    text-align: center;
    padding: 8px 0 16px 0;
    font-size: 24px;
    color: #555;
  }
  .track {
    display: inline-block;
    vertical-align: top;
    text-align: center;
    padding: 12px 1.66%;
    width: 30%;
    font-size: 12px;
    .track-image-container {
      padding: 4px 2%;
      img {
        width: 100%;
      }
    }
  }
}
</style>
