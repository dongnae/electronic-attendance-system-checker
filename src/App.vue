<template>
  <div v-if="!loading"
       style="height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center;">
    <span v-if="isLoggedIn" @click="logout" class="mdi mdi-logout-variant btn btn-logout"></span>
    <Login v-if="!isLoggedIn"/>
    <div v-else class="container">
      <div id="camera" class="camera"></div>
      <div v-if="isDialogOpened" class="desc">
        <div class="desc-container">
          <p style="font-weight: 600; font-size: 1.7rem;">
            {{ dialogMessage }}
          </p>
          <p v-if="num !== ''" class="desc-txt" style="margin-top: 1rem; font-weight: 800; font-size: 1.4rem;">
            {{ name }} ({{ num }})
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Login from "./components/Login";
import axios from "axios";
import {Html5QrcodeScanner} from "html5-qrcode"

export default {
  components: {Login},
  computed: {
    isLoggedIn() {
      return this.$store.getters.isLoggedIn;
    },
    token() {
      return this.$store.getters.token;
    },
  },
  watch: {
    isLoggedIn() {
      this.loading = false;
      setTimeout(() => {
        this.startCamera();
      }, 0);
    }
  },
  methods: {
    logout() {
      if (confirm("로그아웃하시겠습니까?")) {
        localStorage.clear();
        alert("로그아웃하였습니다.");
        location.reload();
      }
    },
    startCamera() {
      let size = Math.min(window.innerHeight, window.innerWidth);
      let html5QrcodeScanner = new Html5QrcodeScanner(
          "camera",
          {
            fps: 20,
            qrbox: {
              width: size / 3,
              height: size / 3
            },
            aspectRatio: 1.0,
            rememberLastUsedCamera: true,
          },
          /* verbose= */ false);
      html5QrcodeScanner.render(jwt => {
        if (this.prev === jwt) return;
        this.prev = jwt;
        axios.post("https://dnhs-contest.nlog.dev/api/checker/verify", JSON.stringify({jwt}), {
          headers: {
            "Content-Type": "application/json",
            Authorization: this.token
          },
        }).then(ret => {
          if (this.isDialogOpened) return;
          let close = () => {
            this.isDialogOpened = false;
            this.dialogMessage = this.prev = this.num = this.name = "";
          }
          let code = ret.data.result;

          this.isDialogOpened = true;
          if (code === -2) {
            this.dialogMessage = "만료된 QR 코드입니다.";
            setTimeout(close, 700);
          } else {
            if (code === 0) {
              let {num, name} = ret.data.result_data;
              this.num = num;
              this.name = name;
              this.dialogMessage = "출석하였습니다.";
            } else if (code === 1) this.dialogMessage = "이미 출석 처리되어 있습니다.";
            setTimeout(close, 1500);
          }
        });
      }, () => {
      });
    }
  },
  data() {
    return {
      loading: true,
      prev: "",
      isDialogOpened: false,
      dialogMessage: "",
      num: "",
      name: "",
    };
  },
  created() {
    let token = localStorage.getItem("token");
    if (typeof token === "string") this.$store.commit("login", token);
    else this.loading = false;
  }
}
</script>

<style>
@import url("https://cdn.jsdelivr.net/npm/reset-css@5.0.1/reset.min.css");
@import url("https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css");
@import url("https://cdn.jsdelivr.net/npm/@mdi/font@6.5.95/css/materialdesignicons.min.css");

@-webkit-keyframes blur-in {
  0% {
    backdrop-filter: blur(0px);
  }
  25% {
    backdrop-filter: blur(0.5px);
  }
  50% {
    backdrop-filter: blur(1px);
  }
  75% {
    backdrop-filter: blur(1.7px);
  }
  100% {
    backdrop-filter: blur(2.5px);
  }
}

* {
  font-family: Pretendard, -apple-system, BlinkMacSystemFont, system-ui, Roboto, 'Helvetica Neue', 'Segoe UI', 'Apple SD Gothic Neo', 'Noto Sans KR', 'Malgun Gothic', sans-serif !important;
  box-sizing: border-box;
}

/* Chrome, Safari, Edge, Opera */
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

/* Firefox */
input[type=number] {
  -moz-appearance: textfield;
}

input {
  border: none;
}

input:focus {
  outline: none;
}

.btn {
  padding: .5rem 1rem;
  border: 1px solid rgba(0, 0, 0, 0.3);
  border-radius: 5px;

  text-align: center;

  cursor: pointer;

  background: none;
  transition: .3s;
}

.btn:hover {
  background: rgba(0, 0, 0, .05);
}

.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.container > * {
  margin-bottom: 1rem;
  width: 100%;
}

.camera {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;

  width: min(100vw, 100vh);
  height: min(100vw, 100vh);
}

.desc {
  position: fixed;
  width: 100vw;
  height: 100vh;
  z-index: 999;

  backdrop-filter: blur(2.5px);
  animation: blur-in .3s;
  background: rgba(0, 0, 0, 0.002);

  display: flex;
  align-items: center;
  justify-content: center;
}

.desc-container {
  width: 80vw;

  padding: 3rem 10rem;

  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;

  background: white;
  border-radius: 5px;
  box-shadow: 2px 2px 20px 3px rgba(0, 0, 0, 0.25);
}

.btn-logout {
  position: absolute;
  right: 1rem;
  top: 1rem;

  color: black;

  border: none;
  padding: 1rem;
}

.desc-container > p {
  text-align: center;
}

.camera > *:nth-child(1), .camera > *:nth-child(3) {
  /* display: none !important; */
}

@media (max-width: 700px) {
  .desc-container {
    padding: 5rem .5rem;
    width: 90vw;
  }
}
</style>
