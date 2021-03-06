<template>
  <section>
    <h2 class="subtitle strong"><strong>Test Details</strong></h2>
    <div class="box">

      <div class="content" v-if="!editMode">
        
        <span class="title is-4">
          <strong>{{ model.name }}</strong>
          <b-icon 
            :icon="model.status === 'ok' ? 'checkbox-marked-circle-outline' : 'alert-circle-outline'" 
            size="is-medium"
            custom-size="mdi-18px"
            :type="model.status === 'ok' ? 'is-success' : 'is-danger'"
          ></b-icon>
        </span>
        <component-status-tags :thresholds="model.thresholds" v-if="model.thresholds"></component-status-tags>
       
        <div>
          <p>
            {{ model.description }}
          </p>
        </div>

      </div>

      <div class="content" v-if="editMode">
        <div class="media">
          <div class="media-left">
            <b-field>
              <b-input placeholder="name" v-model="model.name" required></b-input>
            </b-field>
            <b-field>
              <b-input placeholder="description" v-model="model.description"></b-input>
            </b-field>
            <b-field>
              <b-select placeholder="Threshold" v-model="selectedThreshold" @input="selectedChaged">
                <option
                    v-for="option in metrics"
                    :value="option"
                    :key="option">
                    {{ option }}
                </option>
              </b-select>
              <b-input type="number" min="0" placeholder="0" v-model="selectedThresholdValue"></b-input>
              <p class="control">
                <button class="button is-success" @click="setThreshold">Set</button>
              </p>
            </b-field>
            <b-field label="Key Metric">
              <b-select placeholder="Key Metric" v-model="model.keyMetric">
                  <option>none</option>
                  <option
                      v-for="option in metrics"
                      :value="option" :key="option">
                      {{ option }}
                  </option>
              </b-select>
            </b-field>
            <div class="field">
              <b-tooltip label="Fail the test if last run has any errors" position="is-right" type="is-light">
                <b-switch v-model="model.failOnError" type="is-danger">Fail on error</b-switch>
              </b-tooltip>
            </div>
            <div class="field">
              <b-tooltip label="Fail the test if any of the threshold settings are not met" position="is-right" type="is-light">
                <b-switch v-model="model.failOnThreshold" type="is-danger">Fail on any unmet threshold</b-switch>
              </b-tooltip>
            </div>
            <div class="field">
              <b-tooltip label="Fail the test if key performance metric is not met" position="is-right" type="is-light">
                <b-switch v-model="model.failOnThreshold" type="is-danger">Fail on unmet key metric</b-switch>
              </b-tooltip>
            </div>
          </div>
        </div>
      </div>

      <nav class="level">
        <div class="level-left">
          <a class="level-item" aria-label="reply">
            <button :class="['button', editMode ? 'is-primary' : '']" @click="editClicked">
              <b-icon :icon="editMode ? 'check' : 'pencil'" size="is-small"></b-icon>
              <span>{{ editMode ? 'Save' : 'Edit' }}</span>
            </button>
          </a>
          <a v-if="editMode" class="level-item" aria-label="reply">
            <button class="button" @click="cancelClicked">
              <b-icon icon="cancel" size="is-small"></b-icon>
              <span>Cancel</span>
            </button>
          </a>
        </div>
      </nav>

    </div>

    <b-collapse class="card" v-if="latestRun">
      <div slot="trigger" slot-scope="props" class="card-header">
          <p class="card-header-title">
              Latest Run
          </p>
          <a class="card-header-icon">
              <b-icon
                  :icon="props.open ? 'menu-down' : 'menu-up'">
              </b-icon>
          </a>
      </div>
      <div class="card-content">
        <component-run-detail :run="latestRun"></component-run-detail>
      </div>
    </b-collapse>
    <br />
    <b-collapse class="card" v-if="runs">
    <div slot="trigger" slot-scope="props" class="card-header">
        <p class="card-header-title">
            Change over time
        </p>
        <a class="card-header-icon">
              <b-icon
                  :icon="props.open ? 'menu-down' : 'menu-up'">
              </b-icon>
          </a>
      </div>
      <div class="card-content">
        <component-runs-over-time :runs="runs"></component-runs-over-time>
      </div>
    </b-collapse>
    <br />
    <component-run-list :project-id="projectId" :testId="testId"></component-run-list>
  </section>
</template>

<script>
import axios from 'axios'

import StatusTags from './StatusTags.vue'
import RunDetail from './RunDetail.vue'
import RunsOverTime from './RunsOverTime.vue'
import RunList from './RunList.vue'

export default {
  data() {
    return {
      id: null,
      loading: false,
      editMode: false,
      model: {
        name: '',
        description: '',
        failOnError: false,
        thresholds: null
      },
      selectedThreshold: 'median',
      selectedThresholdValue: 0,
      metrics: ['median', 'mean', '95th', 'fastest', 'slowest', 'RPS'],
      latestRun: null,
      runs: null
    }
  },
  props: {
    projectId: [String, Number],
    testId: [String, Number]
  },
  watch: {
    projectId(newVal, oldVal) {
      this.loadData()
    },
    testId(newVal, oldVal) {
      this.loadData()
    }
  },
  store: ['project', 'test'],
  methods: {
    async loadData() {
      this.loading = true
      try {
        if (!this.test) {
          this.test = await this.$store.fetchTest(this.projectId, this.testId)
        }

        if (!this.latestRun) {
          this.latestRun = await this.$store.fetchLatestRun(this.projectId, this.testId)
        }

        if (!this.runs) {
          this.runs = await this.$store.fetchRuns(this.projectId, this.testId, false, true)
        }

        Object.assign(this.model, this.test)

        this.loading = false
      } catch (e) {
        this.loading = false

        this.$snackbar.open({
          message: e.message,
          type: 'is-danger',
          position: 'is-top'
        })
      }
    },

    async editClicked() {
      if (this.editMode) {
        this.loading = true

        try {
          if (this.model.keyMetric === "none") {
            this.model.keyMetric = ""
          }

          this.test = await this.$store.updateTest(this.projectId, this.model)
          Object.assign(this.model, this.test)

          this.loading = false
        } catch (e) {
          this.loading = false

          this.$snackbar.open({
            message: e.message,
            type: 'is-danger',
            position: 'is-top'
          })

          this.loadData()
        }
      } else {
        this.selectedChaged()
      }

      this.editMode = !this.editMode
    },

    async cancelClicked() {
      await this.loadData()
      this.editMode = false
    },

    setThreshold() {
      if (!this.model.thresholds) {
        this.model.thresholds = {
          mean: { status: 'ok', threshold: 0 },
          median: { status: 'ok', threshold: 0 },
          '95th': { status: 'ok', threshold: 0 },
          fastest: { status: 'ok', threshold: 0 },
          slowest: { status: 'ok', threshold: 0 },
          rps: { status: 'ok', threshold: 0 }
        }
      }

      if (!this.model.thresholds[this.selectedThreshold]) {
        this.model.thresholds[this.selectedThreshold] = { status: 'ok', threshold: 0 }
      }

      const val = parseInt(this.selectedThresholdValue, 10)
      this.model.thresholds[this.selectedThreshold].threshold = val
    },

    selectedChaged() {
      if (!this.model.thresholds) {
        this.selectedThresholdValue = 0
        return
      }

      if (!this.model.thresholds[this.selectedThreshold]) {
        this.selectedThresholdValue = 0
        return
      }

      this.selectedThresholdValue = this.model.thresholds[this.selectedThreshold].threshold || 0
    }
  },
  mounted() {
    this.loadData()

    this.selectedChaged()
  },
  components: {
    'component-status-tags': StatusTags,
    'component-run-detail': RunDetail,
    'component-runs-over-time': RunsOverTime,
    'component-run-list': RunList
  }
}
</script>
