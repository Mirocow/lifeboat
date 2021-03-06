<template>
  <div :class="`tab-${activeTab}`">
    <header>
      <div class="level is-mobile status-bar">
        <div class="level-left">
          <div class="title is-4">
            {{ $store.getters.projectDirName(project.id) }}
            <span :class="{ 'tag': true, 'is-primary': running, 'is-warning': starting }">{{ statusText }}</span>
          </div>
        </div>
        <div class="level-right">
          <div class="level-item">
            <div class="field has-addons">
              <div class="control" v-if="!running && !restarting">
                <div :class="{ dropdown: true, 'is-hoverable': !starting, 'is-right': true }">
                  <div class="dropdown-trigger">
                    <button @click.prevent="start(project.id)" :class="{ button: true, 'is-primary': true, 'is-loading': starting }" aria-haspopup="true" aria-controls="start-button" title="Start">
                      <span class="icon">
                        <i class="fa fa-play"></i>
                      </span>
                      <span class="icon is-small">
                        <i class="fa fa-angle-down" aria-hidden="true"></i>
                      </span>
                    </button>
                  </div>
                  <div class="dropdown-menu" id="start-button" role="menu">
                    <div class="dropdown-content">
                      <a href="#" class="dropdown-item" @click.prevent="start(project.id)">
                        <span class="icon">
                          <i class="fa fa-play"></i>
                        </span>
                        Start
                      </a>
                      <a href="#" class="dropdown-item" @click.prevent="buildAndStart(project.id)">
                        <span class="icon">
                          <i class="fa fa-archive"></i>
                        </span>
                        Build and Start
                      </a>
                    </div>
                  </div>
                </div>
              </div>
              <p class="control" v-if="partiallyRunning && !starting && !stopping">
                <button @click.prevent="restart(project.id)" :class="{ button: true,  'is-loading': restarting}" title="Restart">
                  <span class="icon">
                  <i class="fa fa-refresh"></i>
                  </span>
                </button>
              </p>
              <p class="control" v-if="partiallyRunning">
                <button @click.prevent="stop(project.id)" :class="{ button: true, 'is-loading': stopping}" title="Stop">
                  <span class="icon">
                    <i class="fa fa-stop"></i>
                  </span>
                </button>
              </p>
            </div>
          </div>
        </div>
      </div>
    </header>

    <div class="services" v-show="!missingComposeFile">
      <div class="columns is-mobile is-multiline">
        <div v-for="service in project.services" :key="service" class="column is-one-third">
          <project-service :project="project" :service="service" :container="containerForService(service)" :class="{ active: commandService === service }"></project-service>
        </div>
      </div>
    </div>

    <div class="tabs" v-show="!missingComposeFile">
      <ul>
        <li :class="{'is-active': activeTab === 'logs'}">
          <a href="#" @click.prevent="setActiveTab('logs')">
            <span class="icon is-small">
              <i class="fa fa-eye"></i>
            </span>
            <span>Logs</span>
          </a>
        </li>
        <li :class="{'is-active': activeTab === 'about'}">
          <a href="#" @click.prevent="setActiveTab('about')">
            <span class="icon is-small">
              <i class="fa fa-info-circle"></i>
            </span>
            <span>About</span>
          </a>
        </li>
        <li :class="{'is-active': activeTab === 'commands'}">
          <a href="#" @click.prevent="setActiveTab('commands')">
            <span class="icon is-small">
              <i class="fa fa-play-circle"></i>
            </span>
            <span>Commands</span>
          </a>
        </li>
      </ul>
    </div>

    <div class="tab-area" ref="tabArea" v-show="!missingComposeFile">
      <project-log :project="project" v-show="activeTab === 'logs'"></project-log>
      <project-log-filter :project="project" v-if="activeTab === 'logs' && activeLogFilters.length"></project-log-filter>
      <project-readme :project="project" v-show="activeTab === 'about'"></project-readme>
      <project-commands :project="project" v-show="activeTab === 'commands'"></project-commands>
    </div>

    <div class="notification" v-show="missingComposeFile">
      Lifeboat couldn't find a Docker Compose file in <b>{{ project.dir }}</b>. Please add one and restart Lifeboat.
    </div>

  </div>
</template>

<script>
import { mapGetters, mapActions } from "vuex";
import ProjectService from "@/components/Dashboard/ProjectService";
import ProjectLog from "@/components/Dashboard/ProjectLog";
import ProjectLogFilter from "@/components/Dashboard/ProjectLogFilter";
import ProjectReadme from "@/components/Dashboard/ProjectReadme";
import ProjectCommands from "@/components/Dashboard/ProjectCommands";
import Vue from "vue";
import events from "@/utils/events";
import * as status from "@/utils/project-status";

export default {
  props: ["project"],
  components: {
    ProjectService,
    ProjectLog,
    ProjectLogFilter,
    ProjectReadme,
    ProjectCommands
  },
  methods: {
    containerForService(service) {
      return this.$store.getters
        .containersForProject(this.project.id)
        .find(c => c.service === service);
    },
    setActiveTab(value) {
      this.setTabAreaHeight();
      this.$store.dispatch("updateProjectState", [
        this.project.id,
        "activeTab",
        value
      ]);
    },
    setTabAreaHeight() {
      const height =
        window.innerHeight - this.$refs.tabArea.getBoundingClientRect().top;
      this.$refs.tabArea.style.height = `${height}px`;
    },
    checkForExternalLogs() {
      if ((this.running || this.partiallyRunning) && !this.isLogging) {
        this.$store.dispatch("clearProjectLogs", this.project.id);
        this.$store.dispatch("startProjectLogs", this.project.id);
      }
    },
    ...mapActions({
      start: "startProject",
      stop: "stopProject",
      restart: "restartProject",
      buildAndStart: "buildAndStartProject"
    })
  },
  computed: {
    starting() {
      return this.projectStatus === status.STARTING;
    },
    running() {
      return this.$store.getters.projectRunning(this.project.id);
    },
    partiallyRunning() {
      return this.$store.getters.projectPartiallyRunning(this.project.id);
    },
    stopping() {
      return this.projectStatus === status.STOPPING;
    },
    restarting() {
      return this.projectStatus === status.RESTARTING;
    },
    missingComposeFile() {
      return this.project.missingComposeFile;
    },
    statusText() {
      if (this.starting) {
        return "Starting";
      }

      if (this.running) {
        return "Running";
      }

      return "Stopped";
    },
    activeTab() {
      return this.$store.getters.projectActiveTab(this.project.id);
    },
    projectStatus() {
      return this.$store.state.Project.projects[this.project.id].status;
    },
    isLogging() {
      return !!this.$store.state.Project.projects[this.project.id].isLogging;
    },
    activeLogFilters() {
      return this.projectLogFilters(this.project.id);
    },
    commandService() {
      return this.$store.getters["ProjectCommand/service"](this.project.id);
    },
    ...mapGetters(["activeProject", "projectLogFilters"])
  },
  created() {
    // Do an initial check for external logs after half a second
    setTimeout(this.checkForExternalLogs, 500);

    // Poll the active project every 5s for new logs.
    // This is so unsexy but it's the only way to catch an external project.
    setInterval(this.checkForExternalLogs, 5 * 1000);
  },
  mounted() {
    this.setTabAreaHeight();
    window.addEventListener("resize", this.setTabAreaHeight);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.setTabAreaHeight);
  },
  watch: {
    project(to, from) {
      this.checkForExternalLogs();

      Vue.nextTick(() => {
        this.setTabAreaHeight();
      });
    }
  }
};
</script>

<style lang="scss">
header {
  padding: 1em;
}

.status-bar {
  .level-item:last-child {
    margin-right: 0;
  }
}

.services {
  background-color: var(--color-primary-white);
  padding: 1rem;

  .column {
    padding: 0.35rem;
  }
}

.tabs {
  margin-bottom: 0 !important;
}

.tab-area {
  position: relative;
}
</style>
