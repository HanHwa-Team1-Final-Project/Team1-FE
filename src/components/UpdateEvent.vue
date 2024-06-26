<template>
  <v-container class="update-main">
    <v-form ref="form" lazy-validation>
      <v-text-field
          variant="underlined"
          v-model="eventTitle"
          label="제목"
          :rules="rules.title"
          @blur="$refs.form.validate()"/>
      <v-text-field
          v-model="eventStartDate"
          variant="underlined"
          label="시작일"
          type="datetime-local"
          :rules="[rules.startNotAfterEnd()]"
          @blur="$refs.form.validate()"/>
      <v-text-field
          v-model="eventEndDate"
          variant="underlined"
          label="종료일"
          type="datetime-local"
          :rules="[rules.endNotBeforeStart()]"
          @blur="$refs.form.validate()"/>
      <v-text-field
          v-model="eventPlace"
          variant="underlined"
          label="장소"/>
      <!-- Add Radio buttons for eventMatrix-->
      <v-col cols="12" md="2"><h4>중요도</h4></v-col>
      <v-col cols="12" sm="10">
        <v-container>
          <v-radio-group v-model="eventMatrix"
                         :rules="[value => !!value || '4가지 선택지 중 하나를 선택해주세요']" required>
            
            <v-radio value="Q2">
              <template v-slot:label>
                <div>중요 & 긴급</div>
              </template>
            </v-radio>
            <v-radio value="Q1">
              <template v-slot:label>
                <div>중요 & 긴급하지 않음</div>
              </template>
            </v-radio>
            <v-radio value="Q3">
              <template v-slot:label>
                <div>중요하지 않음 & 긴급</div>
              </template>
            </v-radio>
            <v-radio value="Q4">
              <template v-slot:label>
                <div>중요하지 않음 & 긴급하지 않음</div>
              </template>
            </v-radio>
          </v-radio-group>
        </v-container>
      </v-col>
      <v-textarea
          v-model="eventMemo"
          label="메모"
          auto-grow
          :rules="rules.memo"
          @blur="$refs.form.validate()"/>
      <v-file-input
          v-model="files"
          variant="underlined"
          label="File input"
          placeholder="Upload your documents"
          prepend-icon="mdi-paperclip"
          show-size
          multiple>
        <template v-slot:selection="{ fileNames }">
          <template v-for="fileName in fileNames" :key="fileName">
            <v-chip class="me-2" color="primary" size="small" label>
              {{ fileName }}
            </v-chip>
          </template>
        </template>
      </v-file-input>
      <v-btn @click="updateForm">수정 완료</v-btn>
    </v-form>
    <!-- <p>Event ID: {{ eventId }}</p>
    <p>Event Title: {{ eventTitle }}</p>
    <p>Event Memo: {{ eventMemo }}</p>
    <p>Event StartDate: {{ eventStartDate }}</p>
    <p>Event EndDate: {{ eventEndDate }}</p>
    <p>Event Place: {{ eventPlace }}</p>
    <p>Event Matrix: {{ eventMatrix }}</p>
    <p>Event AlarmYn: {{ eventAlarmYn }}</p>
    <p>Event FileUrl: {{ eventFileUrl }}</p> -->
  </v-container>
</template>

<script>
import {useEventStore} from "@/stores/updateEventStore";
import axiosInstance from "@/axios";

import Swal from 'sweetalert2'

export default {
  data() {
    return {
      eventId: '',
      eventTitle: '',
      eventMemo: '',
      eventStartDate: '',
      eventEndDate: '',
      eventPlace: '',
      eventMatrix: '',
      eventAlarmYn: '',
      eventFileUrl: '',
      rules: {
        title: [
          v => !!v || "제목은 필수 항목입니다!",
          v => (v && v.length <= 20) || "제목은 20자 이내로 작성되어야 합니다."
        ],
        dateTime: [
          v => !!v || "날짜와 시간은 필수 항목입니다!",
          v => !isNaN(Date.parse(v)) || "유효한 날짜와 시간을 입력해주세요."
        ],
        memo: [
          v => (!v || v.length <= 1000) || "메모는 1000자 이내로 작성되어야 합니다."
        ],
        startNotAfterEnd: () => {
          if (this.eventStartDate && this.eventEndDate) {
            return (
                new Date(this.eventStartDate) <= new Date(this.eventEndDate)
            ) || '시작 일시가 종료 일시 이후일 수 없습니다.';
          }
          return true;
        },
        endNotBeforeStart: () => {
          if (this.eventStartDate && this.eventEndDate) {
            return (
                new Date(this.eventEndDate) >= new Date(this.eventStartDate)
            ) || '종료 일시가 시작 일시 이전일 수 없습니다.';
          }
          return true;
        },
      }
    }
  },
  mounted() {
    const eventStore = useEventStore();
    this.eventId = eventStore.currentEvent.id;
    this.eventTitle = eventStore.currentEvent.title;
    this.eventMemo = eventStore.currentEvent.memo;
    this.eventStartDate = eventStore.currentEvent.startDate;
    this.eventEndDate = eventStore.currentEvent.endDate;
    this.eventPlace = eventStore.currentEvent.place;
    this.eventMatrix = eventStore.currentEvent.matrix;
    this.eventAlarmYn = eventStore.currentEvent.alarmYn;
    this.eventFileUrl = eventStore.currentEvent.fileUrl;
  },
  beforeUnmount() {
    // 페이지를 떠날 때 상태를 다시 초기화
    const eventStore = useEventStore();
    eventStore.clearCurrentEvent();
  },
  methods: {
    async updateForm() {
      if (!this.eventTitle) {
        alert("제목을 입력하세요.");
        return;
      }

      if (!this.eventStartDate) {
        alert("시작일을 입력하세요.");
        return;
      }

      if (!this.eventEndDate) {
        alert("종료일을 입력하세요.");
        return;
      }

      if (new Date(this.eventStartDate) > new Date(this.eventEndDate)) {
        alert("시작일은 종료일보다 전이어야 합니다.");
        return;
      }

      // eventRequest 조립
      let eventRequest = {
        title: this.eventTitle,
        memo: this.eventMemo,
        startDate: this.eventStartDate,
        endDate: this.eventEndDate,
        place: this.eventPlace,
        matrix: this.eventMatrix,
        alarmYn: this.eventAlarmYn,
      }
      const eventBlob = new Blob([JSON.stringify(eventRequest)], {type: 'application/json'});
      console.log(eventBlob);

      const formData = new FormData();
      formData.append('eventRequest', eventBlob);

      if (this.files && this.files.length > 0) {
        this.files.forEach(file => {
          formData.append('file', file);
        });
      }

      const TOKEN = localStorage.getItem('accessToken');
      const url = `${process.env.VUE_APP_API_BASE_URL}/api/events/${this.eventId}`;

      try {
        await axiosInstance.patch(url, formData, {
          headers: {
            "Authorization": `Bearer ${TOKEN}`,
            "Content-Type": "multipart/form-data"
          }
        });
        console.log("수정 완료")
        Swal.fire({
            title: '일정이 수정되었습니다.',
            icon: 'success',
            showConfirmButton: true,
            confirmButtonColor: '#3085d6',
            confirmButtonText: '확인',
          }).then((result) => {
            if(result.isConfirmed) {
              this.$router.push({ name: "fullCalendarComponent" });
            }
          })
      } catch (error) {
        console.log(error);
      }
    }
  }
}
</script>
<style>
.update-main {
  margin-top: -17%;
}
</style>