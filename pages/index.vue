<template>
  <div id="gift">
    <div class="bgimg"></div>
    <FreeGift class="bg-back" :freeGiftList="freeGiftList" />
    <PreGift class="bg-back" :preGiftList="preGiftList" />
    <Comment class="bg-back" :commentData="commentData" />
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      roomId: "382233",
      telop: "",
      commentData: [],
      freeGiftList: [],
      preGiftList: [],
      streamData: null,
      socket: null,
      checkStreaming: null,
      startTime: null,
      showFlg: true,
      pon: 0,
    };
  },
  head() {
    return {
      title: "葉月てむ専用コメントビューワー",
    };
  },
  mounted() {
    // パラメータがある場合はテスト
    if (this.$route.query.id != undefined) {
      this.roomId = this.$route.query.id;
    }
    // 疎通確認
    this.checkStreaming = setInterval(() => {
      this.checkLive();
    }, 5000);
  },
  methods: {
    async checkLive() {
      let flg = false;
      let preFlg = false;
      // 配信しているか確認
      await axios
        .get(`${process.env.API_URL}/api/users/${this.roomId}`)
        .then((response) => {
          if (response.data.is_onlive) {
            // プレミアライブ中か？
            if (response.data.premium_room_type == 1) {
              preFlg = true;
            } else {
              flg = true;
              clearInterval(this.checkStreaming);
            }
            this.startTime = response.data.current_live_started_at;
          } else {
            console.log("配信停止中");
          }
        });

      if (preFlg) {
        await axios
          .get(`${process.env.API_URL}/api/users/onlive/${this.roomId}`)
          .then((response) => {
            if (response.data.length != undefined) {
              if (response.data) {
                this.streamData = response.data[0];
                clearInterval(this.checkStreaming);
                // 接続
                this.connectSocket();
              }
            }
          });
      } else if (flg) {
        // 配信情報取得
        await this.getLiveData();
        // 接続
        this.connectSocket();
      }
    },
    async getLiveData() {
      await axios
        .get(`${process.env.API_URL}/api/users/live/${this.roomId}`)
        .then((response) => {
          this.streamData = response.data;
          this.title = response.data.room_name;
        });
    },
    connectSocket() {
      console.log("接続開始");
      // 接続
      this.socket = new WebSocket("wss://online.showroom-live.com");
      // 接続確認
      this.socket.onopen = (e) => {
        this.socket.send("SUB\t" + this.streamData.bcsvr_key);
        console.log("コネクションを開始しました");
      };
      // エラー発生時
      this.socket.onerror = (error) => {
        // alert("エラーが発生しました\nページをリロードしてください");
        location.reload();
      };
      // 疎通確認
      setInterval(() => {
        this.socket.send("PING\tshowroom");
      }, 60000);
      // メッセージ受信
      this.socket.onmessage = (data) => {
        // 死活監視
        if (data.data === "ACK\tshowroom") {
          console.log("死活監視OK");
          return;
        }

        if (data.data === "ERR") {
          // alert("エラーが発生しました\nページをリロードしてください");
          location.reload();
          return;
        }

        // JSON変換
        let getJson = JSON.parse(
          data.data.split("MSG\t" + this.streamData.bcsvr_key)[1]
        );

        if (Object.keys(getJson).length === 9) {
          // コメントログ
          // カウント
          let commentFormat = getJson.cm.replace(/[０-９]/g, (s) => {
            return String.fromCharCode(s.charCodeAt(0) - 0xfee0);
          });
          if (
            Number.isFinite(Number(commentFormat)) &&
            Number(commentFormat) <= 50
          ) {
            // this.getCount(getJson);
          } else {
            this.getComment(getJson);
          }
        } else if (Object.keys(getJson).length === 12) {
          // ギフトログ
          if (getJson.gt == 2) {
            // 投票
            if (Number(getJson.g) > 10000 && Number(getJson.g) <= 10070) {
            } else if (getJson.g == 1601) {
              // 虹星
              this.fallGift(getJson);
            } else {
              // 無料
              this.fallGiftFree(getJson);
              this.giftFree(getJson);
            }
          } else {
            // 有料
            this.fallGift(getJson);
            this.giftPre(getJson);
          }

          // this.fallGift(getJson);
        } else if (Object.keys(getJson).length === 5) {
          // テロップ
          this.telop = getJson.telop;
        } else if (Object.keys(getJson).length === 4) {
          if (getJson.t == 101) {
            this.socket.close();
            // alert("配信が終了しました");
            location.reload();
          }
        } else {
        }
      };
    },
    getComment(commentObj) {
      if (commentObj.cm != undefined) {
        if (commentObj.u == "3699368") {
          // 管理者機能
          let msg = commentObj.cm.split("_");
          if (msg[0] === "g") {
            this.fallAdminGift(commentObj.u, msg[1], msg[2]);
          } else {
            this.commentData = {
              id: commentObj.u,
              name: commentObj.ac,
              comment: commentObj.cm,
              flg: commentObj.ua,
              avatar: commentObj.av,
            };
          }
        } else if (
          commentObj.cm.match(/🤬/) ||
          commentObj.cm.match(/💢/) ||
          commentObj.cm.match(/「いらすとや」/) ||
          commentObj.cm.match(/し、ね/) ||
          commentObj.cm.match(/シ、ネ/) ||
          commentObj.cm.match(/ブ、ス/)
        ) {
          // 荒らし
        } else {
          this.commentData = {
            id: commentObj.u,
            name: commentObj.ac,
            comment: commentObj.cm,
            flg: commentObj.ua,
            avatar: commentObj.av,
          };
        }
      }
    },
    giftFree(giftObj) {
      // 既に存在するか確認
      if (this.freeGiftList.some((e) => e.id == giftObj.u)) {
        for (let i in this.freeGiftList) {
          if (this.freeGiftList[i].id === giftObj.u) {
            this.freeGiftList[i].num += giftObj.n;
            this.freeGiftList[i].gitId = giftObj.g;
            this.freeGiftList[i].name = giftObj.ac;
            this.freeGiftList[i].avatar = giftObj.av;
          }
        }
        // TODO
        let freeGiftData = null;

        this.freeGiftList.some((val, i) => {
          if (val.id == giftObj.u) {
            freeGiftData = val;
            this.freeGiftList.splice(i, 1);
          }
        });

        // 先頭に追加
        this.freeGiftList.unshift(freeGiftData);
      } else {
        this.freeGiftList.unshift({
          id: giftObj.u,
          name: giftObj.ac,
          gitId: giftObj.g,
          num: giftObj.n,
          flg: giftObj.ua,
          avatar: giftObj.av,
        });
      }
    },
    giftPre(giftObj) {
      if (
        this.preGiftList.some(
          (e) => e.id === giftObj.u && e.gitId === giftObj.g
        )
      ) {
        for (let i in this.preGiftList) {
          if (
            this.preGiftList[i].id === giftObj.u &&
            this.preGiftList[i].gitId === giftObj.g
          ) {
            this.preGiftList[i].num += giftObj.n;
            this.preGiftList[i].name = giftObj.ac;
            this.preGiftList[i].avatar = giftObj.av;
          }
        }
        let preGiftData = null;
        this.preGiftList.some((val, i) => {
          if (val.id === giftObj.u && val.gitId === giftObj.g) {
            preGiftData = val;
            this.preGiftList.splice(i, 1);
          }
        });
        this.preGiftList.unshift(preGiftData);
      } else {
        this.preGiftList.unshift({
          id: giftObj.u,
          name: giftObj.ac,
          gitId: giftObj.g,
          num: giftObj.n,
          flg: giftObj.ua,
          avatar: giftObj.av,
        });
      }
    },
    fallGiftFree(gift) {
      // 画面幅を取得
      let width = window.innerWidth;
      let height = window.innerHeight;

      // ギフトの数分ループ
      for (let i = 0; i < gift.n; i++) {
        // 要素のID
        let id = `gift_${gift.u}_${gift.g}_${i}`;
        // ギフト画像の要素を作成
        let giftImgElement = document.createElement("img");
        giftImgElement.classList.add("bg-top");
        // 画像を設定
        giftImgElement.src = `https://image.showroom-cdn.com/showroom-prod/assets/img/gift/${gift.g}_s.png`;

        // IDを設定
        giftImgElement.setAttribute("id", id);
        // 配置位置を設定
        giftImgElement.style.position = "absolute";
        giftImgElement.style.top = "-25px"; // 画面外に配置
        giftImgElement.style.left = this.getRandomNum(10, width - 70) + "px"; // ランダムに配置
        // ギフト要素を画面に追加
        document.getElementById("gift").append(giftImgElement);

        // 動きを追加
        // 動かす要素IDを指定
        TweenMax.to(`#${id}`, {
          duration: this.getRandomNum(2, 5), // 2秒～5秒の間で移動
          rotation: this.getRandomNum(90, 720), // 回転角度
          y: height - 60, // 落ちる高さ
          onComplete: () => {
            document.getElementById(id).remove(); // 終わったら要素を削除
          },
        });
      }
    },
    fallGift(gift) {
      // 画面幅を取得
      let width = window.innerWidth;
      let height = window.innerHeight;

      // ギフトの数分ループ
      for (let i = 0; i < gift.n; i++) {
        // 要素のID
        let id = `gift_${gift.u}_${gift.g}_${i}`;
        // ギフト画像の要素を作成
        let giftImgElement = document.createElement("img");
        giftImgElement.classList.add("bg-top");
        // 画像を設定
        giftImgElement.src = `https://image.showroom-cdn.com/showroom-prod/assets/img/gift/${gift.g}_s.png`;
        giftImgElement.style.width = "100px";
        // IDを設定
        giftImgElement.setAttribute("id", id);
        // 配置位置を設定
        giftImgElement.style.position = "absolute";
        giftImgElement.style.top = "-25px"; // 画面外に配置
        giftImgElement.style.left = this.getRandomNum(10, width - 70) + "px"; // ランダムに配置
        // ギフト要素を画面に追加
        document.getElementById("gift").append(giftImgElement);

        // 動きを追加
        // 動かす要素IDを指定
        TweenMax.to(`#${id}`, {
          duration: this.getRandomNum(2, 5), // 2秒～5秒の間で移動
          rotation: this.getRandomNum(90, 720), // 回転角度
          y: height - 60, // 落ちる高さ
          onComplete: () => {
            document.getElementById(id).remove(); // 終わったら要素を削除
          },
        });
      }
    },
    fallAther(userId, img, num, size) {
      // 画面幅を取得
      let width = window.innerWidth;
      let height = window.innerHeight;

      // ギフトの数分ループ
      for (let i = 0; i < num; i++) {
        // 要素のID
        let id = `pon_${userId}_${i}`;
        // ギフト画像の要素を作成
        let giftImgElement = document.createElement("img");
        giftImgElement.classList.add("bg-top");
        // 画像を設定
        giftImgElement.src = require(`@/assets/image/${img}.png`);
        giftImgElement.style.width = `${size}px`;
        // IDを設定
        giftImgElement.setAttribute("id", id);
        // 配置位置を設定
        giftImgElement.style.position = "absolute";
        giftImgElement.style.top = "-25px"; // 画面外に配置
        giftImgElement.style.left = this.getRandomNum(10, width - 70) + "px"; // ランダムに配置
        // ギフト要素を画面に追加
        document.getElementById("gift").append(giftImgElement);

        // 動きを追加
        // 動かす要素IDを指定
        TweenMax.to(`#${id}`, {
          duration: this.getRandomNum(2, 5), // 2秒～5秒の間で移動
          rotation: this.getRandomNum(90, 720), // 回転角度
          y: height - 60, // 落ちる高さ
          onComplete: () => {
            document.getElementById(id).remove(); // 終わったら要素を削除
          },
        });
      }
    },
    fallAdminGift(userId, gid, num) {
      // 画面幅を取得
      let width = window.innerWidth;
      let height = window.innerHeight;

      // ギフトの数分ループ
      for (let i = 0; i < num; i++) {
        // 要素のID
        let id = `gift_${userId}_${i}`;
        // ギフト画像の要素を作成
        let giftImgElement = document.createElement("img");
        giftImgElement.classList.add("bg-top");
        // 画像を設定
        giftImgElement.src = `https://image.showroom-cdn.com/showroom-prod/assets/img/gift/${gid}_s.png`;
        giftImgElement.style.width = "100px";
        // IDを設定
        giftImgElement.setAttribute("id", id);
        // 配置位置を設定
        giftImgElement.style.position = "absolute";
        giftImgElement.style.top = "-25px"; // 画面外に配置
        giftImgElement.style.left = this.getRandomNum(10, width - 70) + "px"; // ランダムに配置
        // ギフト要素を画面に追加
        document.getElementById("gift").append(giftImgElement);

        // 動きを追加
        // 動かす要素IDを指定
        TweenMax.to(`#${id}`, {
          duration: this.getRandomNum(2, 5), // 2秒～5秒の間で移動
          rotation: this.getRandomNum(90, 720), // 回転角度
          y: height - 60, // 落ちる高さ
          onComplete: () => {
            document.getElementById(id).remove(); // 終わったら要素を削除
          },
        });
      }
    },
    getRandomNum(min, max) {
      min = Math.ceil(min);
      max = Math.floor(max);
      return Math.random() * (max - min + 1) + min;
    },
  },
};
</script>

<style>
#gift {
  background-color: white;
  width: 100vw;
  height: 100vh;
}
.bg-top {
  z-index: 20 !important;
}
.bgimg {
  width: 100vw;
  height: 100vh;
  position: absolute;
  z-index: 10;
  background-image: url("~/assets/image/bg.png");
}
.bg-back {
  z-index: 0;
}
</style>