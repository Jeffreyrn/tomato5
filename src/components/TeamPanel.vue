<template>
  <section class="teamPanel">
    <div class="teamPanel-wrapper" transition="fade" v-if="teamData">
      <p class="title">
        Your team working on tomatoes:
        <!-- <span>{{ teamData.info.name }}</span>     -->
        <span class="expander" v-on:click="showForm()">+</span>
      </p>
      <div class="teamManager" transition="expand" v-show="teamForm.isShow">
        <p class="switch" v-show="false">
          <span class="selector" v-on:click="teamForm.isJoin = true" v-bind:class="{ 'selected': teamForm.isJoin }">Join Team</span>
          <span class="spacer">or</span>
          <span class="selector" v-on:click="teamForm.isJoin = false" v-bind:class="{ 'selected': !teamForm.isJoin }">My Team</span>
        </p>
        <div class="theForm">
          <div class="join" transition="slide_left" v-show="teamForm.isJoin">
            <p class="field">
              <span>Invite Code: </span>
              <input type="text" name="name" value="" v-model="teamForm.inviteCode">
            </p>
            <p>
              <a v-on:click="joinTeam(teamForm.inviteCode)">Join Team</a>
            </p>
          </div>
          <div class="create" transition="slide_right" v-show="!teamForm.isJoin">
            <p class="field">
              <span class="">Name: </span>
              <input type="text" value="" v-model="teamForm.name">
            </p>
            <p class="field">
              <span class="">Invite Code: </span>
              <input type="text" value="" placeholder="6-12 charactors" v-model="teamForm.inviteCode">
            </p>
            <p class="tips">
              Others will use this code to join your team.
            </p>
            <p>
              <a v-on:click="createTeam(teamForm.inviteCode, teamForm.name)">Save</a>
            </p>
          </div>
          <p class="info"></p>
        </div>
      </div>
      <div class="members">
        <member v-for="(uid, member) in teamData.members"
                v-bind:tasks="member.tasks"
                v-bind:user-status="member.userStatus"
                v-bind:user-info="member.userInfo"
                v-bind:update-time="member.updateTime">
        </member>
      </div>
    </div>
  </section>
</template>

<script>
import moment from 'moment';
import Member from './Member';
import database from '../database';
import auth from '../auth';
import { tasks as defaultTasks } from '../model';

const teamData = null;

const teamForm = {
  inviteCode: '',
  name: '',
  isShow: false,
  isJoin: true,
};

const showForm = function showForm() {
  this.teamForm.isShow = !this.teamForm.isShow;
};

let watchRef = null;

const publishToTeam = function publishToTeam() {
  // database.save('test/a', { b: 2 }).then(() => {
  //   console.log('published');
  // });

  return database.save(`teams/${this.userStatus.currentTeam}/members/${auth.getUser().uid}`, {
    rule: 'member',
    userInfo: this.userInfo,
    userStatus: this.userStatus,
    tasks: this.tasks,
    updateTime: moment(),
  }).then(() => {
    console.log('published');
  });
};

const joinTeam = function joinTeam(inviteCode) {
  if (inviteCode) {
    this.userStatus.currentTeam = inviteCode;
    if (watchRef) {
      watchRef.off();
    }

    this.publishToTeam().then(() => {
      this.watchTeam();
    });
  }
};

const createTeam = function createTeam(inviteCode, name) {
  if (inviteCode && name) {
    database.save(`teams/${inviteCode}`, {
      info: { inviteCode, name },
      members: [],
    }).then(() => {
      console.log('Team created');
      return this.joinTeam(inviteCode);
    });
  }
};

const changeTeamName = function changeTeamName(inviteCode, name) {
  database.save(`teams/${inviteCode}/info`, { inviteCode, name }).then(() => {
    console.log('Name changed');
  });
};

const processTeamData = function processTeamData(data) {
  if (data && data.members) {
    Object.keys(data.members).forEach((uid) => {
      const member = data.members[uid];

      member.tasks = member.tasks.filter((task) => (task.createTime
          && moment(task.createTime).format('YYYYMMDD') === moment().add(0, 'd').format('YYYYMMDD'))
      );

      member.tasks = member.tasks.length > 0 ? member.tasks : defaultTasks;
    });
  }

  return data;
};

const watchTeam = function watchTeam() {
  watchRef = database.watch(`teams/${this.userStatus.currentTeam}`, (snapshot) => {
    this.teamData = processTeamData(snapshot.val());
  });
};

const init = function init() {
  this.joinTeam(this.userStatus.currentTeam);
};

export default {
  props: ['userInfo', 'userStatus', 'tasks'],
  data() {
    return { teamData, teamForm };
  },
  events: {
    publish: publishToTeam,
  },
  created: init,
  methods: { publishToTeam, watchTeam, joinTeam, createTeam, changeTeamName, showForm },
  components: { Member },
};
</script>

<style lang="scss" scoped>
@import "../base";

.teamPanel {
  .title {
    margin: 30px 0 0px 0;;
    text-align: left;
    line-height: 30px;
  }
  .expander {
    color: $blue;
    line-height: 30px;
    display: inline-block;
    padding: 0px 10px;
    cursor: pointer;
    user-select: none;
  }

  .teamManager {
    margin: 0px 0;
    .switch {
      margin: 30px 0 0px 0;
      user-select: none;
      .selector {
        padding: 5px 10px;
        border-radius: 5px;
        transition: all .3s;
        color: $blue;
        cursor: pointer;
      }
      .spacer {
        margin: 0 10px;
      }
      .selected {
        background-color: $green;
        color: white;
        cursor: default;
      }
    }

    .theForm {
      position: relative;
    }

    .field {
      span {
        margin-right: 10px;
      }
    }

    .tips {
      color: #ccc;
    }
  }
}

.slide_left-transition {
  transition: all .5s;
  width: 100%;
  position: absolute;;
  overflow: hidden;
  top: 0;
  left: 0;
}

.slide_left-enter, .slide_left-leave {
  left: -500px;
  opacity: 0;
}

.slide_right-transition {
  transition: all .5s;
  width: 100%;
  position: absolute;
  overflow: hidden;
  left: 0;
  top: 0;
}

.slide_right-enter, .slide_right-leave {
  left: 700px;
  opacity: 0;
}

/* always present */
.expand-transition {
  transition: all .3s ease;
  height: 100px;
  overflow: hidden;
}

/* .expand-enter defines the starting state for entering */
/* .expand-leave defines the ending state for leaving */
.expand-enter, .expand-leave {
  height: 0;
  opacity: 0;
}
</style>