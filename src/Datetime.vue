<template>
  <div class="vdatetime">
    <slot name="before"></slot>
    <div class="input-control">
      <input class="vdatetime-input material-input"
             :class="[(value || value===0)?'active':'', isValid?'':'no-validate', wideText ? 'wide-text' : '']"
             :style="inputStyle"
             :id="inputId"
             type="text"
             :value="inputValue"
             v-bind="$attrs"
             v-on="$listeners"
             @click="open"
             @focus="open">
      <label class="material-label" :class="value ? 'active':''" :for="_uid">
        <span>
          {{label}}
          <span v-if="required" class="required">*</span>
        </span>
      </label>
      <span class="bar"></span>
      <div class="input-error" :class="isValid?'':'error'">{{validMsg}}</div>
    </div>
    <input v-if="hiddenName" type="hidden" :name="hiddenName" :value="value" @input="setValue">
    <slot name="after"></slot>
    <transition-group name="vdatetime-fade" tag="div">
      <div key="overlay" v-if="isOpen" class="vdatetime-overlay" @click.self="cancel"></div>
      <datetime-popup
          key="popup"
          v-if="isOpen"
          :type="type"
          :datetime="popupDate"
          :phrases="phrases"
          :use12-hour="use12Hour"
          :hour-step="hourStep"
          :minute-step="minuteStep"
          :min-datetime="popupMinDatetime"
          :max-datetime="popupMaxDatetime"
          @confirm="confirm"
          @cancel="cancel"
          :auto="auto"
          :week-start="weekStart"
          :flow="flow"
          :title="title">
        <template slot="button-cancel__internal" slot-scope="scope">
          <slot name="button-cancel" v-bind:step="scope.step">{{ phrases.cancel }}</slot>
        </template>
        <template slot="button-confirm__internal" slot-scope="scope">
          <slot name="button-confirm" v-bind:step="scope.step">{{ phrases.ok }}</slot>
        </template>
      </datetime-popup>
    </transition-group>
  </div>
</template>

<script>
  import { DateTime } from 'luxon';
  import DatetimePopup from './DatetimePopup';
  import { datetimeFromISO, startOfDay, weekStart } from './util';

  export default {
    components: {
      DatetimePopup
    },

    inheritAttrs: false,

    props: {
      value: {
        type: String
      },
      label: {
        type: String,
        default: ''
      },
      required: {
        type: Boolean,
        default: false
      },
      wideText: {
        type: Boolean,
        default: false
      },
      isValid: {
        type: Boolean,
        default: true
      },
      validMsg: {
        type: String,
        default: null
      },
      valueZone: {
        type: String,
        default: 'UTC'
      },
      inputId: {
        type: String,
        default: ''
      },
      inputClass: {
        type: [Object, Array, String],
        default: ''
      },
      inputStyle: {
        type: [Object, Array, String],
        default: ''
      },
      hiddenName: {
        type: String
      },
      zone: {
        type: String,
        default: 'local'
      },
      format: {
        type: [Object, String],
        default: null
      },
      type: {
        type: String,
        default: 'date'
      },
      phrases: {
        type: Object,
        default() {
          return {
            cancel: 'Cancel',
            ok: 'Ok'
          };
        }
      },
      use12Hour: {
        type: Boolean,
        default: false
      },
      hourStep: {
        type: Number,
        default: 1
      },
      minuteStep: {
        type: Number,
        default: 1
      },
      minDatetime: {
        type: String,
        default: null
      },
      maxDatetime: {
        type: String,
        default: null
      },
      auto: {
        type: Boolean,
        default: false
      },
      weekStart: {
        type: Number,
        default() {
          return weekStart();
        }
      },
      flow: {
        type: Array
      },
      title: {
        type: String
      }
    },

    data() {
      return {
        isOpen: false,
        datetime: datetimeFromISO(this.value)
      };
    },
    watch: {
      value(newValue) {
        this.datetime = datetimeFromISO(newValue);
      }
    },

    created() {
      this.emitInput();
    },

    computed: {
      inputValue() {
        let {format} = this;

        if (!format) {
          switch (this.type) {
          case 'date':
            format = DateTime.DATE_MED;
            break;
          case 'time':
            format = DateTime.TIME_24_SIMPLE;
            break;
          case 'datetime':
          case 'default':
            format = DateTime.DATETIME_MED;
            break;
          default:
            format = DateTime.DATETIME_MED;
          }
        }

        if (typeof format === 'string') {
          return this.datetime ? DateTime.fromISO(this.datetime).setZone(this.zone).toFormat(format) : '';
        }
        return this.datetime ? this.datetime.setZone(this.zone).toLocaleString(format) : '';
      },
      popupDate() {
        return this.datetime ? this.datetime.setZone(this.zone) : this.newPopupDatetime();
      },
      popupMinDatetime() {
        return this.minDatetime ? DateTime.fromISO(this.minDatetime).setZone(this.zone) : null;
      },
      popupMaxDatetime() {
        return this.maxDatetime ? DateTime.fromISO(this.maxDatetime).setZone(this.zone) : null;
      }
    },

    methods: {
      emitInput() {
        let datetime = this.datetime ? this.datetime.setZone(this.valueZone) : null;

        if (datetime && this.type === 'date') {
          datetime = startOfDay(datetime);
        }
        this.$emit('input', datetime ? datetime.toISO() : '');
      },
      open(event) {
        event.target.blur();
        this.isOpen = true;
      },
      close() {
        this.isOpen = false;
        this.$emit('close');
      },
      confirm(datetime) {
        this.datetime = datetime.toUTC();
        this.emitInput();
        this.close();
      },
      cancel() {
        this.close();
      },
      newPopupDatetime() {
        let datetime = DateTime.utc().setZone(this.zone).set({ seconds: 0, milliseconds: 0 });

        if (this.popupMinDatetime && datetime < this.popupMinDatetime) {
          datetime = this.popupMinDatetime.set({ seconds: 0, milliseconds: 0 });
        }

        if (this.popupMaxDatetime && datetime > this.popupMaxDatetime) {
          datetime = this.popupMaxDatetime.set({ seconds: 0, milliseconds: 0 });
        }

        if (this.minuteStep === 1) {
          return datetime;
        }

        const roundedMinute = Math.round(datetime.minute / this.minuteStep) * this.minuteStep;

        if (roundedMinute === 60) {
          return datetime.plus({ hours: 1 }).set({ minute: 0 });
        }

        return datetime.set({ minute: roundedMinute });
      },
      setValue(event) {
        this.datetime = datetimeFromISO(event.target.value);
        this.emitInput();
      }
    }
  };
</script>

<style lang="scss">
  @import "../../../assets/css/variables";

.vdatetime-fade-enter-active,
.vdatetime-fade-leave-active {
  transition: opacity .4s;
}

.vdatetime-fade-enter,
.vdatetime-fade-leave-to {
  opacity: 0;
}

.vdatetime-overlay {
  z-index: 999;
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  background: rgba(0, 0, 0, 0.5);
  transition: opacity .5s;
}
.vdatetime {
  .required {
    color: #ff5733;
    margin-left: -2px;
  }

  .wide-text {
    @include respond-to(medium-mobile-only) {
      margin-top: 1.5rem;
    }
  }

  .input-control {
    position: relative;
    width: 100%;
    margin-top: 10px;
    margin-bottom: 20px;
  }

  i {
    -webkit-user-select: none; /* Chrome all / Safari all */
    -moz-user-select: none; /* Firefox all */
    -ms-user-select: none; /* IE 10+ */
    user-select: none; /* Likely future */
  }

  .material-input::placeholder {
    transition: 1s;
    opacity: 0;
  }

  .material-input:focus.material-input::placeholder {
    opacity: 1;

  }

  .material-input {
    background: inherit;
    width: 100%;
    font-size: 16px;
    display: inline-block;
    border: none;
    padding-bottom: 4px;
    margin-right: 5px;
    border-bottom: 1px solid #757575;
    -webkit-border-radius: 0;
    -webkit-appearance: none;
    margin-bottom: 0 !important;

  }

  .material-input:focus {
    border-bottom: 1px solid #757575;
    color: unset !important;
    outline: none;
  }

  .material-input:disabled {
    background: #ececec2b !important;

  }

  .material-input:disabled:hover {
    cursor: not-allowed;
  }

  .material-input:disabled ~ .material-label {
    color: #a0a0a0 !important;

  }

  .material-label {
    position: absolute;
    pointer-events: none;
    margin: 0;
    top: 0;
    left: 0;
    font-size: 16px;

    transition: transform .7s cubic-bezier(.44, 1.4, .44, 1), color .7s
     cubic-bezier(.44, 1.4, .44, 1), width .7s cubic-bezier(.44, 1.4, .44, 1);
    transition-property: transform, color, width;
    transition-duration: 0.7s, 0.7s, 0.7s;
    transition-timing-function: cubic-bezier(0.44, 1.4, 0.44, 1),
     cubic-bezier(0.44, 1.4, 0.44, 1), cubic-bezier(0.44, 1.4, 0.44, 1);
    transition-delay: 0s, 0s, 0s;

    transform-origin: 0 0;
  }

  .material-input:focus ~ .material-label,
  .material-input:-webkit-autofill ~ .material-label,
  .active ~ .material-label {
    transform: translateY(-1.04em) scale(.875) perspective(100px) translateZ(.001px);
    width: 133.33%;
    color: var(--primary-color, #1e81f0);
  }


  .bar {
    z-index: 40; /* level 4 */
    position: relative;
    display: block;

  }

  .bar:before, .bar:after {
    content: '';
    height: 2px;
    width: 0;
    bottom: 0;
    position: absolute;
    background: var(--primary-color, #1e81f0);
    /*background: #1e81f0;*/
    transition: 0.5s cubic-bezier(0.25, 0.8, 0.5, 1);
    -moz-transition: 0.5s cubic-bezier(0.25, 0.8, 0.5, 1);
    -webkit-transition: 0.5s cubic-bezier(0.25, 0.8, 0.5, 1);
  }

  .bar:before {
    left: 50%;
  }

  .bar:after {
    right: 50%
  }

  /* active state */
  .material-input:focus ~ .bar:before, .material-input:focus ~ .bar:after {
    background: var(--primary-color, #1e81f0);
    width: 50%;
  }

  .input-error {
    position: relative;
    -webkit-transform: scaleY(1) translateY(-10px);
    transform: scaleY(1) translateY(-10px);
    z-index: -10; /* level -2 */
    height: 15px;
    opacity: 0;
    color: #ff5733;
    transition: 0.7s cubic-bezier(0.25, 0.8, 0.5, 1);

  }

  .material-input.no-validate {
    color: #ff0300;
    border-bottom: 1px solid #ff0300 !important;

  }

  .material-input.no-validate ~ .bar:before, .material-input.no-validate ~ .bar:after {
    background: #ff5733;

  }

  .material-input.no-validate ~ .material-label {
    color: #ff5733
  }

  .material-input.no-validate ~ .input-error {
    -webkit-transform: scaleY(1) translateY(0);
    transform: scaleY(1) translateY(0);

    -webkit-transition: 0.7s cubic-bezier(0.25, 0.8, 0.5, 1);
    transition: 0.7s cubic-bezier(0.25, 0.8, 0.5, 1);
    z-index: 10; /* level 1 */
    opacity: 1;
    height: auto;
    animation-name: bounce;
    animation-duration: .5s;
    /*width: auto;*/

  }

  .material-input.no-validate + .material-label {
    color: #ff5733 !important;

  }

  .material-input.no-validate + .material-label > div {
    animation-duration: .5s;
    animation-name: bounce;
  }

  @keyframes bounce {
    0% {
      -webkit-transform: translateX(0px);
      transform: translateX(0px);
    }
    37% {
      -webkit-transform: translateX(5px);
      transform: translateX(5px);
    }
    55% {
      -webkit-transform: translateX(-5px);
      transform: translateX(-5px);
    }
    73% {
      -webkit-transform: translateX(4px);
      transform: translateX(4px);
    }
    82% {
      -webkit-transform: translateX(-4px);
      transform: translateX(-4px);
    }
    91% {
      -webkit-transform: translateX(2px);
      transform: translateX(2px);
    }
    96% {
      -webkit-transform: translateX(-2px);
      transform: translateX(-2px);
    }
    100% {
      -webkit-transform: translateX(0px);
      transform: translateX(0px);
    }
  }
}
</style>
