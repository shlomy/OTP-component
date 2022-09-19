<template>
  <div>
    <v-row id="otp-steps">
      <v-col class="text-center black--text step" v-if="step1">
        <p class="display-24 black--text">קוד אימות יישלח למספר</p>
        <p class="display-24 mt-3 mb-2 font-weight-bold">{{ userPhone }}</p>
        <v-btn
          class="mt-2"
          rounded
          color="primary"
          @click="sendOtp()"
          :loading="loading"
          :min-width="'150'"
          :height="'40'"
        >
          לשליחת הקוד לחצו כאן
        </v-btn>

        <nuxt-link
          :to="'/personal'"
          class="mt-5 d-block text-center secondary--text display-14"
        >
          זה לא המספר שלי
        </nuxt-link>
        <!-- 
        <v-checkbox
          color="orange"
          v-model="allowSms"
          class="mx-2 my-0"
          :label="'מאשר קבלת מסרונים'"
        ></v-checkbox> -->
      </v-col>

      <v-col class="text-center black--text step" v-if="step2">
        <p class="display-24 black--text">
          <!-- <span><v-icon color="primary" size="">mdi-greater-than</v-icon></span> -->
          שלחנו לך קוד אימות למספר
        </p>
        <p class="display-24 mt-3 mb-2 font-weight-bold">{{ userPhone }}</p>
        <v-form
          v-model="otpModel"
          ref="otpForm"
          @submit.prevent="authenticateOtp()"
          class="mx-auto"
          :disabled="loading"
        >
          <v-text-field
            v-model="otpCode"
            type="text"
            name="input_otp"
            required
            dense
            outlined
            solo
            color="primary"
            class="primary--text mx-auto mb-5 mt-3 text-center"
            :rules="passwordRules"
            @copy.prevent
            @cut.prevent
            maxlength="6"
            label="הקוד שקיבלת"
            id="opt-field"
            :hide-details="true"
          ></v-text-field>
          <v-btn
            type="submit"
            dense
            rounded
            color="primary"
            :loading="loading"
            :disabled="!otpModel"
            large
            :min-width="'150'"
            :height="'40'"
          >
            המשך
          </v-btn>
        </v-form>

        <p class="mt-5 display-14 pointer" @click="startOverOpt()">
          לא קיבלתי , שלחו שוב
        </p>
      </v-col>

      <v-col class="text-center step" v-if="step3">
        <p class="text-h6 black--text mb-5">Success</p>

        <v-btn @click="close()">close</v-btn>
      </v-col>

      <v-col class="red--text py-0 text-center error-msg">
        <span v-if="error">{{ error }}</span>
      </v-col>
    </v-row>
    <v-btn
      color="primary font-bold darken-1 "
      text
      @click="close()"
      id="otp-close"
    >
      <span class="text-h5">x</span>
    </v-btn>
  </div>
</template>


<script>
import CurrentUser from '@/components/mixins/CurrentUser';
import { AUTHENTICATE_OTP, SEND_OTP } from '@/lib/api.js';
import { passwordRules } from '@/lib/rules.js';
export default {
  mixins: [CurrentUser],
  props: {
    userPhone: {
      type: String,
      required: true,
      default: () => {
        '';
      },
    },
  },
  data() {
    return {
      step1: true, // send otp to the user
      step2: false, // submit code
      step3: false, // success
      error: null, // error message
      allowSms: false, // checkbox to accept terms
      loading: false, // loading state of code
      otpCode: '', // otp code field
      otpModel: false, // otp form model
      passwordRules, // otp code field regex validation rules
      errorsList: {
        '-3040': 'המספר כבר קיים במערכת, כדאי לנסות שוב.', // Sender Id is already Allowed or whitelist not required
        '-3015': 'משהו לא עובד. שננסה מההתחלה?', // You not allowed to use this function
        '-220': ' אופס, זה לא הקוד ששלחנו לך. נא לבדוק ולנסות שוב.', //  OTP code not match
        '-370': 'אופס פיספסת והקוד כבר לא בתוקף. כדאי לנסות שוב', //  OTP is Expired
        general: 'משהו לא עובד. שננסה מההתחלה?', // general error uncaught error statu ID
      },
    };
  },
  methods: {
    close() {
      this.$emit('closeDialog', false);
      this.startOverOpt();
    },
    //reset process to step 1
    startOverOpt() {
      this.step1 = true;
      this.step2 = false;
      this.step3 = false;
      this.loading = false;
      this.otpCode = '';
      this.error = null;
    },
    // display steps api errors
    throwOtpError(description, id, StatusDescription) {
      this.error = description;
      this.loading = false;
      this.otpCode = '';
      throw id + '-' + StatusDescription;
    },
    async sendOtp() {
      this.error = false;
      this.loading = true;
      let payload = {
        PhoneNumber: this.userPhone,
      };
      try {
        let { data } = await this.$axios.post(`${SEND_OTP}`, payload);

        if (data.IsError) {
          throw data.Errors;
        }
        // this.step1 = false;
        // this.step2 = true;
        // this.loading = false;

        // Response from shamir contains:
        // {
        // "StatusId": 1,
        // "StatusDescription": "Success",
        // "DetailedDescription": "",
        // "FunctionName": "api/v2/SMS/Whitelist/SendOtp",
        // "RequestId":"lhdfsdkjfhsdlhfsdk",
        // "Data": {
        // "Sent": true,
        // "SenderId": "0529999999",
        // "UserId": 1234
        // }
        // }
        // List of Possible Statuses:
        // 1 – OK, OTP sent
        // -3040 – Sender Id is already Allowed or whitelist not required
        // -3015 – You not allowed to use this function

        if (data.StatusId === '1') {
          this.step1 = false;
          this.step2 = true;
          this.loading = false;
        } else if (data.StatusId === '-3040') {
          this.throwOtpError(
            this.errorsList[data.StatusId],
            data.StatusId,
            data.StatusDescription
          );
        } else if (data.StatusId === '-3015') {
          this.throwOtpError(
            this.errorsList[data.StatusId],
            data.StatusId,
            data.StatusDescription
          );
        } else {
          this.throwOtpError('משהו לא עובד. שננסה מההתחלה?', data.StatusId);
          this.throwOtpError(
            this.errorsList['general'],
            data.StatusId,
            data.StatusDescription
          );
        }

        // StatusDescription - success
      } catch (e) {
        console.log('send Otp failed', e);
        // this.startOverOpt();
      }
    },
    async authenticateOtp() {
      this.loading = true;
      this.error = false;

      let payload = {
        PhoneNumber: this.userPhone,
        OtpCode: this.otpCode,
      };
      try {
        let { data } = await this.$axios.post(`${AUTHENTICATE_OTP}`, payload);

        if (data.IsError) {
          throw data.Errors;
        }

        // Response from shamir
        // {
        // "StatusId": 1,
        // "StatusDescription": "Success",
        // "DetailedDescription": "",
        // "FunctionName": "api/v2/SMS/AuthenticateOtp",
        // "RequestId":"lhdfsdkjfhsdlhfsdk",
        // "Data": {
        // "SenderId": "0529999999",
        // "CustomerId": 1234
        // }
        // }
        // List of Possible Statuses:
        // 1 – OK, Whitelist Sender completed successfully
        // -220 – OTP code not match
        // -370 – OTP is Expired

        if (data.StatusId === '1') {
          //1 – OK, Whitelist Sender completed successfull
          this.step1 = false;
          this.step2 = false;
          this.step3 = true;
          this.loading = false;
        } else if (data.StatusId === '-220') {
          //-220 – OTP code not match
          this.throwOtpError(
            this.errorsList[data.StatusId],
            data.StatusId,
            data.StatusDescription
          );
        } else if (data.StatusId === '-370') {
          this.throwOtpError(
            this.errorsList[data.StatusId],
            data.StatusId,
            data.StatusDescription
          );
        } else {
          this.throwOtpError(
            this.errorsList['general'],
            data.StatusId,
            data.StatusDescription
          );
        }
      } catch (e) {
        console.log('send Otp failed', e);
        // this.startOverOpt();
      }
    },
  },
};
</script>
<style lang="sass">
#otp-close
  position: absolute
  top: 5px
  left: 0
#otp-steps
  align-content: space-around
  justify-content: center
  flex-direction: column
  min-height: 250px
  .error-msg
    position: absolute
    bottom: 10px
  form
    display: flex
    justify-content: space-between
    flex-direction: column

  label
    text-align: center
    left: 0 !important
    right: 0 !important
    margin: 0 auto
  input
    text-align: center
  .step
    display: flex
    flex-direction: column
    align-items: center
    justify-content: center
</style>