<template>
      <div class="MatcMainMenu MatcMainMenuPublic" id="mainMenu">
      <div class="MatcMainMenuHeader">
        <div id="menuBar" class="MatcMenuBar">
          <div class="container visible-md-block visible-lg-block">

            <div class="row" v-if="user && user.role !== 'guest'">
              <div class="col-md-7">
                <a class="MatcMainMenuItem" href="#/">{{$t('header.my-prototypes')}}</a>
              </div>
              <div class="col-md-5 MatcRight">
                <a class="MatcMainMenuItem" href="#/my-account.html">{{$t('header.my-account')}}</a>
                <a class="MatcMainMenuItem MatcRightMenuLast" @click="logout">{{$t('header.logout')}}</a>
              </div>
            </div> <!-- Logged in user -->
          </div> <!-- Desktop -->
          <div class="visible-sm-block visible-xs-block">
             <div class="row" v-if="user && user.role !== 'guest'">
                <div class="col-md-12">
                  <a class="MatcMainMenuItem" href="#/my-apps.html">{{$t('header.my-prototypes')}}</a>
                </div>
             </div>
          </div>
        </div>
      </div>
    </div>
</template>

<script>

import Services from 'services/Services'
import Logger from 'common/Logger'
import hash from "dojo/hash";

export default {
  name: "Header",
  mixins: [],
  props: ['user'],
  data: function() {
    return {
    }
  },
  watch: {
    'user' (v) {
      this.logger.log(6, 'watch', 'user >> ' + v.email)
      this.user = v
    }
  },
  components: {
  },
  methods: {
    logout() {
      this.logger.log(2, "logout", "entry");
      Services.getUserService().logout()
      this.$emit('logout', Services.getUserService().GUEST)
      hash("/", true);
    }
  },
  async mounted() {
    this.logger = new Logger('Header')
    this.logger.log(7, 'mounted', 'exit >> ' + this.user.email)
  }
}
</script>

