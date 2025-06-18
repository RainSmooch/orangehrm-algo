<!--
  /**
  * OrangeHRM is a comprehensive Human Resource Management (HRM) System that captures
  * all the essential functionalities required for any enterprise.
  * Copyright (C) 2006 OrangeHRM Inc., http://www.orangehrm.com
  *
  * OrangeHRM is free software: you can redistribute it and/or modify it under the terms of
  * the GNU General Public License as published by the Free Software Foundation, either
  * version 3 of the License, or (at your option) any later version.
  *
  * OrangeHRM is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
  * without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
  * See the GNU General Public License for more details.
  *
  * You should have received a copy of the GNU General Public License along with OrangeHRM.
  * If not, see <https://www.gnu.org/licenses/>.
  */
  -->

<template>
  <oxd-form :loading="isLoading" @submit-valid="onSave">
    <oxd-form-row>
      <oxd-grid :cols="4" class="orangehrm-full-width-grid">
        <template v-if="attendanceRecord.previousRecord">
          <oxd-grid-item
            :class="
              !attendanceRecord.previousRecord.note ? '--span-column-2' : ''
            "
          >
            <oxd-input-group :label="$t('attendance.punched_in_time')">
              <oxd-text type="subtitle-2">
                {{ previousAttendanceRecordDate }} -
                {{ previousAttendanceRecordTime }}
                <oxd-text
                  tag="span"
                  class="orangehrm-attendance-punchedIn-timezone"
                >
                  {{ `(GMT ${previousRecordTimezone})` }}
                </oxd-text>
              </oxd-text>
            </oxd-input-group>
          </oxd-grid-item>

          <oxd-grid-item v-if="attendanceRecord.previousRecord.note">
            <oxd-input-group :label="$t('Punched in Location')">
              <oxd-text type="subtitle-2">
                {{ attendanceRecord.previousRecord.note }}
              </oxd-text>
            </oxd-input-group>
          </oxd-grid-item>
        </template>

        <oxd-grid-item class="--offset-row-2">
          <date-input
            :key="attendanceRecord.time"
            v-model="attendanceRecord.date"
            :label="$t('general.date')"
            :rules="rules.date"
            :disabled="!isEditable"
            required
          />
        </oxd-grid-item>

        <oxd-grid-item class="--offset-row-2">
          <oxd-input-field
            v-model="attendanceRecord.time"
            :label="$t('general.time')"
            :disabled="!isEditable"
            :rules="rules.time"
            type="time"
            :placeholder="$t('attendance.hh_mm')"
            required
          />
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>

    <oxd-grid v-if="isTimezoneEditable" :cols="2">
      <oxd-grid-item>
        <timezone-dropdown v-model="attendanceRecord.timezone" required />
      </oxd-grid-item>
    </oxd-grid>

    <oxd-form-row>
      <oxd-grid :cols="4" class="orangehrm-full-width-grid">
        <oxd-grid-item class="--span-column-2">
          <oxd-input-group :label="$t('Location')">
            <div
              id="geolocation-map"
              style="
                height: 250px;
                width: 100%;
                border-radius: 8px;
                margin-bottom: 8px;
              "
            ></div>

            <oxd-text
              tag="p"
              v-if="locationAddress"
              style="font-weight: 600; margin-bottom: 4px"
            >
              {{ locationAddress }}
            </oxd-text>

            <oxd-text tag="p" class="orangehrm-attendance-punchedIn-timezone">
              {{ locationInfo }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>
    <oxd-divider />
    <oxd-form-actions>
      <required-text />
      <submit-button
        :label="
          !attendanceRecordId ? $t('attendance.in') : $t('attendance.out')
        "
      />
    </oxd-form-actions>
  </oxd-form>
</template>

<script>
/* global L */
import {
  required,
  validDateFormat,
  shouldNotExceedCharLength,
} from '@/core/util/validation/rules';
import {
  parseTime,
  parseDate,
  formatTime,
  formatDate,
  guessTimezone,
  setClockInterval,
  getStandardTimezone,
} from '@/core/util/helper/datefns';
import {promiseDebounce} from '@ohrm/oxd';
import useLocale from '@/core/util/composable/useLocale';
import {APIService} from '@ohrm/core/util/services/api.service';
import useDateFormat from '@/core/util/composable/useDateFormat';
import {reloadPage, navigate} from '@/core/util/helper/navigation';
import TimezoneDropdown from '@/orangehrmAttendancePlugin/components/TimezoneDropdown.vue';

const attendanceRecordModal = {
  date: null,
  time: null,
  note: null,
  timezone: null,
  previousRecord: null,
};

export default {
  name: 'RecordAttendance',
  components: {
    'timezone-dropdown': TimezoneDropdown,
  },
  props: {
    isEditable: {
      type: Boolean,
      default: false,
    },
    isTimezoneEditable: {
      type: Boolean,
      default: false,
    },
    attendanceRecordId: {
      type: Number,
      default: null,
    },
    employeeId: {
      type: Number,
      default: null,
    },
    date: {
      type: String,
      default: null,
    },
  },
  setup(props) {
    const apiPath = props.employeeId
      ? `/api/v2/attendance/employees/${props.employeeId}/records`
      : '/api/v2/attendance/records';
    const http = new APIService(window.appGlobal.baseUrl, apiPath);
    const {jsDateFormat, userDateFormat, timeFormat, jsTimeFormat} =
      useDateFormat();
    const {locale} = useLocale();
    return {
      http,
      locale,
      timeFormat,
      jsTimeFormat,
      jsDateFormat,
      userDateFormat,
    };
  },
  data() {
    return {
      isLoading: false,
      attendanceRecord: {...attendanceRecordModal},
      rules: {
        date: [
          required,
          validDateFormat(this.userDateFormat),
          promiseDebounce(this.validateDate, 500),
        ],
        time: [required, promiseDebounce(this.validateDate, 500)],
        // note: [shouldNotExceedCharLength(250)], // <-- --- ATURAN VALIDASI NOTE DIHAPUS ---
      },
      previousRecordTimezone: null,
      // --- Data untuk Geolocation ---
      latitude: null,
      longitude: null,
      locationInfo: this.$t('attendance.getting_location'),
      isLocationReady: false,
      map: null,
      locationAddress: '',
    };
  },
  computed: {
    previousAttendanceRecordDate() {
      if (!this.attendanceRecord?.previousRecord) return null;
      return formatDate(
        parseDate(this.attendanceRecord.previousRecord.userDate),
        this.jsDateFormat,
        {locale: this.locale},
      );
    },
    previousAttendanceRecordTime() {
      if (!this.attendanceRecord?.previousRecord) return null;
      return formatTime(
        parseTime(
          this.attendanceRecord.previousRecord.userTime,
          this.timeFormat,
        ),
        this.jsTimeFormat,
      );
    },
  },
  beforeMount() {
    this.isLoading = true;
    // set default timezone
    if (this.isTimezoneEditable) {
      const tz = guessTimezone();
      this.attendanceRecord.timezone = {
        id: tz.name,
        label: tz.label,
        _name: tz.name,
        _offset: tz.offset,
      };
    }

    // fetch and set attendance record on initial load
    this.setCurrentDateTime()
      .then(() => {
        // then set record date/time every minute
        !this.date &&
          !this.isEditable &&
          setClockInterval(this.setCurrentDateTime, 60000);
        let url = '/api/v2/attendance/records/latest';
        if (this.employeeId) {
          url = `/api/v2/attendance/records/latest?empNumber=${this.employeeId}`;
        }
        return this.attendanceRecordId
          ? this.http.request({method: 'GET', url})
          : null;
      })

      .then((response) => {
        if (response) {
          const {data} = response.data;
          this.attendanceRecord.previousRecord = data.punchIn;
        }
      })
      .then(() => {
        this.previousRecordTimezone = getStandardTimezone(
          this.attendanceRecord.previousRecord?.offset,
        );
      })
      .finally(() => {
        this.isLoading = false;
      });
  },
  mounted() {
    this.initGeolocation();
  },
  methods: {
    initGeolocation() {
      const mapContainer = document.getElementById('geolocation-map');
      if (!mapContainer) {
        setTimeout(this.initGeolocation, 100);
        return;
      }

      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          this.showPosition,
          this.showError,
          {enableHighAccuracy: true, timeout: 10000, maximumAge: 0},
        );
      } else {
        this.locationInfo = this.$t('attendance.geolocation_not_supported');
        mapContainer.style.display = 'none';
      }
    },

    // --- FUNGSI INI DIMODIFIKASI SECARA TOTAL ---
    showPosition(position) {
      this.latitude = position.coords.latitude;
      this.longitude = position.coords.longitude;
      this.isLocationReady = true;

      this.map = L.map('geolocation-map').setView(
        [this.latitude, this.longitude],
        17,
      );
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution:
          '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      }).addTo(this.map);

      const marker = L.marker([this.latitude, this.longitude]).addTo(this.map);
      const apiUrl = `https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${this.latitude}&lon=${this.longitude}`;

      fetch(apiUrl)
        .then((response) => response.json())
        .then((data) => {
          let locationString = '';
          if (data.display_name) {
            this.locationAddress = data.display_name;
            this.locationInfo = `Coordinate: ${this.latitude.toFixed(
              5,
            )}, ${this.longitude.toFixed(5)}`;

            // Gabungkan alamat dan koordinat menjadi satu string.
            locationString = `${
              data.display_name
            } (Lat: ${this.latitude.toFixed(5)}, Lon: ${this.longitude.toFixed(
              5,
            )})`;
            marker
              .bindPopup(`<b>You Are Here</b><br>${this.locationAddress}`)
              .openPopup();
          } else {
            // Jika API alamat gagal, gunakan koordinat saja.
            this.locationInfo = `Location found: ${this.latitude.toFixed(
              5,
            )}, ${this.longitude.toFixed(5)}`;
            locationString = `Location at Lat: ${this.latitude.toFixed(
              5,
            )}, Lon: ${this.longitude.toFixed(5)}`;
            marker.bindPopup(`<b>You Are Here</b>`).openPopup();
          }

          // Masukkan string lokasi yang sudah jadi ke dalam properti 'note'.
          this.attendanceRecord.note = locationString;
        })
        .catch((error) => {
          console.error('Error fetching address:', error);
          const locationString = `Failed to get location name. Coords: ${this.latitude.toFixed(
            5,
          )}, ${this.longitude.toFixed(5)}`;
          this.locationInfo = locationString;
          // Jika error, catat error tersebut sebagai note.
          this.attendanceRecord.note = locationString;
          marker.bindPopup(`<b>You Are Here</b>`).openPopup();
        });
    },

    showError(error) {
      const mapContainer = document.getElementById('geolocation-map');
      switch (error.code) {
        case error.PERMISSION_DENIED:
          this.locationInfo = this.$t(
            'attendance.geolocation_permission_denied',
          );
          break;
        case error.POSITION_UNAVAILABLE:
          this.locationInfo = this.$t('attendance.geolocation_unavailable');
          break;
        case error.TIMEOUT:
          this.locationInfo = this.$t('attendance.geolocation_timeout');
          break;
        default:
          this.locationInfo = this.$t('attendance.geolocation_unknown_error');
          break;
      }
      if (mapContainer) mapContainer.style.display = 'none';
    },

    // --- FUNGSI INI DIMODIFIKASI ---
    onSave() {
      this.isLoading = true;
      const timezone = guessTimezone();

      // Objek ini sekarang jauh lebih sederhana dan hanya mengirim 'note' yang sudah berisi data lokasi.
      const dataToSave = {
        date: this.attendanceRecord.date,
        time: this.attendanceRecord.time,
        timezoneOffset:
          this.attendanceRecord.timezone?._offset ?? timezone.offset,
        timezoneName: this.attendanceRecord.timezone?.id ?? timezone.name,
        note: this.attendanceRecord.note,
      };

      this.http
        .request({
          method: this.attendanceRecordId ? 'PUT' : 'POST',
          data: dataToSave,
        })
        .then(() => {
          return this.$toast.saveSuccess();
        })
        .then(() => {
          this.employeeId
            ? navigate('/attendance/viewAttendanceRecord', undefined, {
                employeeId: this.employeeId,
                date: this.date,
              })
            : reloadPage();
        });
    },
    setCurrentDateTime() {
      return new Promise((resolve, reject) => {
        this.http
          .request({method: 'GET', url: '/api/v2/attendance/current-datetime'})
          .then((res) => {
            const {utcDate, utcTime} = res.data.data;
            const currentDate = parseDate(
              `${utcDate} ${utcTime} +00:00`,
              'yyyy-MM-dd HH:mm xxx',
            );
            this.attendanceRecord.date =
              this.date ?? formatDate(currentDate, 'yyyy-MM-dd');
            this.attendanceRecord.time = formatDate(currentDate, 'HH:mm');
            resolve();
          })
          .catch((error) => reject(error));
      });
    },
    validateDate() {
      if (!this.attendanceRecord.date || !this.attendanceRecord.time) {
        return true;
      }
      if (parseDate(this.attendanceRecord.date) === null) {
        return true;
      }
      const tzOffset = (new Date().getTimezoneOffset() / 60) * -1;
      return new Promise((resolve) => {
        this.http
          .request({
            method: 'GET',
            url: `/api/v2/attendance/${
              this.attendanceRecordId ? 'punch-out' : 'punch-in'
            }/overlaps`,
            params: {
              date: this.attendanceRecord.date,
              time: this.attendanceRecord.time,
              timezoneOffset:
                this.attendanceRecord.timezone?._offset ?? tzOffset,
              empNumber: this.employeeId,
            },
            validateStatus: (status) => {
              return (status >= 200 && status < 300) || status == 400;
            },
          })
          .then((res) => {
            const {data, error} = res.data;
            if (error) {
              return resolve(error.message);
            }
            return data.valid === true
              ? resolve(true)
              : resolve(this.$t('attendance.overlapping_records_found'));
          });
      });
    },
  },
};
</script>

<style src="./record-attendance.scss" lang="scss" scoped></style>
