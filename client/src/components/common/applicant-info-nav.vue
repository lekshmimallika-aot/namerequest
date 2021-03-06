<template>
  <v-col cols="5" class="text-right py-0">
    <v-btn x-large
           :id="`submit-back-btn-${isValid}`"
           class="mr-3"
           v-if="showBack"
           @click="back()">
      {{ backText }}
    </v-btn>
    <v-btn x-large
           @click="next()"
           :disabled="!isValid"
           :loading="isloadingSubmission"
           :id="`submit-continue-btn-${isValid}`">
      {{ nextText }}
    </v-btn>
  </v-col>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator'
import newReqModule from '@/store/new-request-module'
import paymentModule from '@/modules/payment'
import timerModule from '@/modules/vx-timer'
import { sleep } from '@/plugins/sleep'

@Component({})
export default class ApplicantInfoNav extends Vue {
  @Prop(Boolean) isValid: boolean
  private isloadingSubmission: boolean = false

  get backText () {
    if (this.editMode) {
      return 'Previous'
    }
    return 'Back'
  }
  get editMode () {
    return newReqModule.editMode
  }
  get nextText () {
    if (this.submissionTabNumber === 3) {
      if (this.editMode) {
        return 'Submit Changes'
      }
      return 'Review and Confirm'
    }
    if (this.editMode) {
      return 'Next'
    }
    return 'Continue'
  }
  get nrId () {
    return newReqModule.nrId
  }
  get nrState () {
    return newReqModule.nrState
  }
  get showBack () {
    if (this.submissionTabNumber < 2) {
      return false
    }
    if (this.submissionTabNumber === 2) {
      return (this.type === 'examination' || this.nrState === 'DRAFT')
    }
    if (this.submissionTabNumber === 3) {
      return true
    }
    return false
  }
  get submissionTabNumber (): number {
    return newReqModule.submissionTabNumber
  }
  get type () {
    return newReqModule.submissionType
  }
  back () {
    newReqModule.mutateSubmissionTabNumber(this.submissionTabNumber - 1)
  }
  async next () {
    if (this.submissionTabNumber === 3) {
      this.isloadingSubmission = true
      await this.submit()
      return
    }
    newReqModule.mutateSubmissionTabNumber(this.submissionTabNumber + 1)
  }

  /** Submits an edited NR or a new name submission. */
  async submit () {
    // FUTURE: fix error handling in case of newReqModule (app or api) error (#5899)
    const { nrId } = this
    if (this.editMode) {
      // stop timer first so it doesn't expire during network requests
      timerModule.stopTimer({ id: this.$EXISTING_NR_TIMER_NAME })
      if (await newReqModule.patchNameRequests()) {
        if (await newReqModule.checkinNameRequest()) {
          newReqModule.mutateDisplayedComponent('Success')
          await sleep(1000)
          await this.fetchNr(+nrId)
        }
      }
    } else {
      let request
      if (!nrId) {
        // FUTURE: does a timer have to be stopped here?
        request = await newReqModule.postNameRequests('draft')
      } else {
        if (!this.editMode && ['COND-RESERVE', 'RESERVED'].includes(this.nrState)) {
          request = await newReqModule.getNameRequest(nrId)
          if (request?.stateCd === 'CANCELLED') {
            newReqModule.setActiveComponent('Timeout')
            this.isloadingSubmission = false
            // FUTURE: does a timer have to be stopped here before returning?
            return
          }
        }
        request = await newReqModule.putNameReservation(nrId)
        timerModule.stopTimer({ id: this.$NR_COMPLETION_TIMER_NAME })
      }
      if (request) await paymentModule.togglePaymentModal(true)
    }
    this.isloadingSubmission = false
  }

  async fetchNr (nrId: number): Promise<void> {
    const nrData = await newReqModule.getNameRequest(nrId)
    await newReqModule.loadExistingNameRequest(nrData)
  }
}
</script>
