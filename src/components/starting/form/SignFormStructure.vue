<template>
  <transition-group
    tag="div"
    @before-enter="beforeEnter"
    @enter="enter"
    @leave="leave"
    appear
    mode="out-in"
  >
    <sign-form-input
      v-for="(data, index) in userData"
      :data-index="index"
      :key="index"
      :inputText="data.inputText"
      :userData="userData"
      :userInput="data.userInput"
      :formSent="formSent"
      :ok="data.ok"
      @sendToFire="checkFileds"
    ></sign-form-input>

    <ion-button
      v-if="btnOn"
      color="light"
      class="signbtn"
      :data-index="4"
      @click.prevent="formSent++"
      >{{ text[1] }}
    </ion-button>

    <ion-button
      v-if="btnOn && signOption === 'sign-in'"
      class="forgot"
      fill="clear"
      :data-index="5"
      @click="forgotPass = true"
      >שכחתי סיסמה
    </ion-button>

    <p v-if="errMsg && btnOn && !forgotPass" :data-index="4">{{ errMsg }}</p>
    <div class="arrow" :data-index="6" v-if="btnOn" >
      {{ text[0] }}
      <arrow-animation :deg="0"></arrow-animation>
    </div>
  </transition-group>

  <transition @before-enter="beforeEnter" @enter="enter" @leave="leave" appear
    ><sign-form-forgot
  
      v-if="forgotPass"
      @closeForgot="forgotPass = false"
      :oldEmail="userData[0].userInput"
    ></sign-form-forgot
  ></transition>
</template>

<script>
import SignFormInput from "./SignFormInput.vue";
import SignFormForgot from "./SignFormForgot.vue";
import ArrowAnimation from "../intro/ArrowAnimation.vue";
import { defineComponent } from "vue";
import { IonButton } from "@ionic/vue";
import gsap from "gsap";
import {
  getAuth,
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  updateProfile,
} from "firebase/auth";

export default defineComponent({
  name: "SignFormStructure",
  components: { SignFormInput, SignFormForgot, ArrowAnimation, IonButton },
  props: ["signOption"],
  emits: ["signOptChanged"],

  data() {
    return {
      inputsText: [],
      fullUserData: [
        { input: "email", inputText: "המייל שלי", userInput: "", ok: true },
        { input: "password", inputText: "הסיסמה שלי", userInput: "", ok: true },
        { input: "verify", inputText: "אימות הסיסמה", userInput: "", ok: true },
        {
          input: "fullName",
          inputText: "אתם יכולים לקרוא לי...",
          userInput: "",
          ok: true,
        },
      ],
      userData: [],
      errMsg: "",
      btnOn: false,
      text: [],
      formSent: 0,
      forgotPass: false,
    };
  },

  setup() {
    const beforeEnter = (el) => {
      el.style.opacity = 0;
      el.style.transform = "translateY(2vh)";
    };

    const enter = (el, done) => {
      gsap.to(el, {
        opacity: 1,
        y: 0,
        duration: 0.8,
        onComplete: done,
        delay: el.dataset.index * 0.25,
      });
    };

    const leave = (el, done) => {
      gsap.to(el, {
        opacity: 0,
        y: 10,
        duration: 0.8,
        onComplete: done,
        delay: (3 - el.dataset.index) * 0.5,
      });
    };

    return { beforeEnter, enter, leave };
  },

  methods: {
    checkFileds() {
      let email = this.userData[0].userInput;
      let password = this.userData[1].userInput;
      let atpos = email.indexOf("@");

      if (email == "") {
        this.catchProblem("no-email");
        return;
      }

      if (atpos < 1 || email.lastIndexOf(".") - atpos < 2) {
        this.catchProblem("auth/invalid-email");
        return false;
      }

      if (password == "") {
        this.catchProblem("no-password");
        return;
      }

      if (this.signOption === "sign-up") {
        let name = this.userData[3].userInput;
        if (password.length < 6) {
          this.catchProblem("auth/weak-password");
          return;
        }

        if (this.userData[2].userInput == "") {
          this.catchProblem("no-ver-password");
          return;
        }

        if (password != this.userData[2].userInput) {
          this.catchProblem("diff-password");
          return;
        }

        if (name == "") {
          this.catchProblem("no-name");
          return;
        }
      }
      return this.sendToFire();
    },

    sendToFire() {
      if (this.signOption === "sign-in") {
        signInWithEmailAndPassword(
          getAuth(),
          this.userData[0].userInput,
          this.userData[1].userInput
        )
          .then((data) => {
            console.log("successfully sign in", data);
            this.$router.push({ name: "Home" });
          })
          .catch((error) => {
            this.catchProblem(error.code);
          });
      } else {
        createUserWithEmailAndPassword(
          getAuth(),
          this.userData[0].userInput,
          this.userData[1].userInput
        )
          .then((data) => {
            console.log("regidtered!", data);
          })
          .then(() => {
            updateProfile(getAuth().currentUser, {
              displayName: this.userData[3].userInput,
            }).then(() => {
              this.$router.push({ name: "Home" });
            });
          })
          .catch((error) => {
            this.catchProblem(error.code);
          });
      }
    },

    catchProblem(error) {
      console.log(error);
      for (let i in this.userData) {
        this.userData[i].ok = true;
      }
      let notOk = [];
      switch (error) {
        case "no-email":
          this.errMsg = "מה עם המייל?";
          notOk.push(0);
          break;
        case "auth/invalid-email":
          this.errMsg = " אתם בטוחים שהמייל הזה קיים ?";
          notOk.push(0);
          break;
        case "auth/email-already-in-use":
          this.errMsg = "כתובת המייל כבר תפוסה";
          notOk.push(0);
          break;
        case "no-password":
          this.errMsg = "מה עם סיסמה?";
          notOk.push(1);
          break;
        case "auth/weak-password":
          this.errMsg = "הסיסמה לא עומדת ברף - לפחות 6 תווים";
          notOk.push(1);
          break;
        case "auth/invalid-password":
          this.errMsg = "הסיסמה לא תקינה";
          notOk.push(1);
          break;
        case "auth/wrong-password":
          this.errMsg = "הסיסמה שגויה";
          notOk.push(1);
          break;
        case "diff-password":
          this.errMsg = "הסיסמאות לא תואמות";
          notOk.push(1);
          break;
        case "no-ver-password":
          this.errMsg = "הקלד את הסיסמה שנית";
          notOk.push(2);
          break;
        case "no-name":
          this.errMsg = "נשמח לדעת מה שמך";
          notOk.push(3);
          break;
        case "auth/user-not-found":
          this.errMsg = "המייל לא קיים במערכת";
          notOk.push(0);
          break;
        default:
          this.errMsg = "משהו השתבש נסה שוב";
          notOk = this.signOption === "sign-in" ? [0, 1] : [0, 1, 2, 3];
          break;
      }
      notOk.forEach((i) => (this.userData[i].ok = false));
    },
  },

  watch: {
    signOption: {
      immediate: true,
      handler() {
        this.btnOn = !this.btnOn;
        this.errMsg = "";
        this.$emit("signOptChanged");

        this.userData = [];
        let dataRequired = this.signOption === "sign-in" ? 2 : 4;
        for (let i = 0; i < dataRequired; i++) {
          this.userData.push(this.fullUserData[i]);
        }

        if (this.signOption === "sign-in") {
          this.text = ["עדיין אין לי משתמש", "התחבר"];
        } else {
          this.text = ["אני כבר רשום", "הירשם"];
        }
        setTimeout(() => {
          this.btnOn = true;
        }, 500);
      },
    },
  },
});
</script>

<style scoped>

div {
  display: flex;
  flex-direction: column;
  width: 55vw;
}

ion-button.signbtn {
  --background: var(--ion-color-light);
  padding-top: 0.5vh !important;
  padding-bottom: 0.5vh !important;
  padding-left: 3vh !important;
  padding-right: 3vh !important;
  margin: 3vh;
  margin-bottom: 0vh;
  font-weight: 600;
  font-size: 2.5vh;
  --border-radius: 2vh;
}

.arrow {
  position: absolute;
  right: 0;
  bottom: 2vh;
  height: 10vh;
  width: 100vw;
  display: flex;
  align-items: center;
  flex-direction: column;
  font-size: 2.3vh;
  color: var(--ion-color-secondary);
}
.arrow > div {
  width: 15vw;
}

p {
  width: 50vw;
  text-align: center;
  font-size: 5vw;
  margin-left: auto;
  margin-right: auto;
}

.forgot {
  text-decoration: underline;
  font-size: 2vh;
  margin-bottom: 2vh;
}

.forgot:active {
  color: var(--ion-color-secondary);
}

.btn-box {
  display: flex;
  flex-direction: column;
}

</style>
