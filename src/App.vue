<template>
  <div id="app" v-infinite-scroll="handleInfiniteOnLoad" infinite-scroll-distance="10">
    <a-modal v-model="visible" title="下载" @ok="handleOk">
      <p>{{dialogContent}}</p>
    </a-modal>
    <a-input-search placeholder="输入音乐名" :loading="loading" enter-button @search="onSearch" />
    <a-list :data-source="musicData" style="margin-bottom:16px">
      <a-list-item slot="renderItem" slot-scope="item">
        <a-list-item-meta :description="item.ARTIST">
          <a slot="title">{{ item.NAME }}</a>
          <a-avatar shape="square" size="large" slot="avatar" :src="'https://img2.kuwo.cn/star/albumcover/'+item.web_albumpic_short" />
        </a-list-item-meta>

        <a-button :type="item1[3]!='flac'?'primary':'danger'" :key="i" size="small" class="vbb" v-for="(item1,i) in item.MINFO" @click="onDownload(item.MUSICRID.substring(6),item1[2])">
          {{item1[3].toUpperCase()}}
        </a-button>
      </a-list-item>
      <div v-if="loading && !busy && this.hasMore" style="text-align: center;">
        <a-spin />
      </div>
      <div v-if="!this.hasMore" style="text-align: center;">
        ---已经到底了---
      </div>
    </a-list>
  </div>
</template>

<script>
import axios from "axios"
import infiniteScroll from 'vue-infinite-scroll';
export default {
  directives: { infiniteScroll },
  data() {
    return {
      visible: false,
      dialogContent: "",
      musicData: [],
      loading: false,
      busy: false,
      keyword: "",
      page: 0,
      hasMore: true
    };
  },
  methods: {
    handleOk(e) {
      this.visible = false;
      require('electron').shell.openExternal(this.dialogContent);
    },
    fetchData(kw, callback) {
      if (!this.hasMore) {
        return
      }
      this.loading = true
      setTimeout(() => {
        let res = axios.get("http://search.kuwo.cn/r.s?client=kt&all=" + kw + "&pn=" + this.page + "&rn=15&uid=&vipver=1&show_copyright_off=1&newver=1&ft=music&cluster=0&strategy=2012&encoding=utf8&rformat=json&vermerge=1&mobi=1&issubtitle=1")
        res.then((v) => {
          this.loading = false
          if (v.data.abslist == undefined) {
            this.$message.error("无更多结果")
            this.hasMore = false
            return
          }

          v.data.abslist.forEach(e => {
            var str = e.MINFO
            e.MINFO = []
            var reg = /level:(.*?),bitrate:(.*?),format:(.*?),size:(.*?)b/gi
            var res = str.match(reg)
            for (let index = 0; index < res.length; index++) {
              e.MINFO.push(reg.exec(str))
            }
          });


          callback(v.data.abslist)
          this.page++
        })
      }, 800);
    },
    handleInfiniteOnLoad() {
      if (this.keyword == "") {
        return
      }
      this.loading = true;

      this.fetchData(this.keyword, (data) => {
        data.forEach(element => {
          this.musicData.push(element)
        });
      })
    },
    onDownload(rid, br) {
      const KW = require('./kwDES')
      var url = "https://nmobi.kuwo.cn/mobi.s?f=kuwo&q=" + KW.encryptQuery("user=0&prod=kwplayer_ar_6.8.9.0&corp=kuwo&vipver=6.8.9.0&source=kwplayer_ar_6.8.9.0_lenovo02.apk&p2p=1&type=convert_url2&br=" + br + "&rid=" + rid)
      // console.log(url);
      axios.get(url).then((v) => {
        this.dialogContent = v.data.split("\r\n")[2].replaceAll("url=", "");
        this.visible = true;
      })
    },
    onSearch(v) {
      this.hasMore = true
      this.page = 0
      if (v == "") {
        this.$message.warning("请输入搜索关键字")
        return
      }
      this.keyword = v
      this.musicData = []
      this.fetchData(v, (data) => {
        data.forEach(element => {
          this.musicData.push(element)
        });
        console.log(this.musicData);
      })


    }
  },
};
</script>
<style>
.vbb {
  margin: 2px;
}
#app {
  padding: 16px;
}
</style>
