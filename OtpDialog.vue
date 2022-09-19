<template>
  <v-row justify="center">
    <v-dialog v-model="dialog" persistent max-width="500" width="500">
      <!-- <template v-slot:activator="{ on, attrs }">
        <v-btn color="primary" dark v-bind="attrs" v-on="on">Open Dialog</v-btn>
      </template> -->
      <v-card class="align-self-center">
        <!-- <v-card-title class="text-h5">
          Use Google's location service?
        </v-card-title> -->
        <v-card-text class="pt-5">
          <!-- {{ $auth.loggedIn }}
          <br /> -->

          <SendOtp
            @closeDialog="dialog = false"
            v-if="$auth.loggedIn"
            :userPhone="userPhone"
          />
        </v-card-text>
        <!-- <v-card-actions> -->
        <!-- <v-spacer></v-spacer> -->

        <!-- <v-btn
          color="primary font-bold darken-1 "
          text
          @click="dialog = false"
          id="otp-close"
        >
          <span class="text-h5">x</span>
        </v-btn> -->
        <!-- </v-card-actions> -->
      </v-card>
    </v-dialog>
  </v-row>
</template>





<script>
import CurrentUser from '@/components/mixins/CurrentUser';
import SendOtp from '@/components/OTP/SendOtp.vue';

export default {
  mixins: [CurrentUser],
  components: { SendOtp },
  data() {
    return {
      dialog: false,
      isUserPhoneValid: false,
    };
  },
  computed: {
    // require key in user model if number is not authenticated
    userPhone() {
      if (this.CurrentUser && this.CurrentUser.Profile.Phone) {
        return this.CurrentUser.Profile.Phone;
      } else {
        return null;
      }
    },
  },
  mounted() {
    this.$nuxt.$on('openOTP', this.openOTP);
  },
  methods: {
    async openOTP() {
      // display otp if required by user profile
      if (this.userPhone && !this.isUserPhoneValid) {
        console.log('number is not valid - open opt ditalog ');
        await this.$auth.fetchUser();
        this.dialog = true;
      } else {
        console.log('number is already valid');
      }
    },
  },
  beforeDestroy() {
    this.$nuxt.$off('openOTP');
  },
};
</script>

<style lang="sass" >
</style>

