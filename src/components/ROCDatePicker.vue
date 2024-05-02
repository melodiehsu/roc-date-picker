<template>
  <div>
    <div class="date-picker-container">
      <button
        class="input-container"
        type="button"
        @click="toggleCalender"
      >
        <div class="input-icon">
          <CalendarDayIcon />
        </div>

        <input
          class="date-picker-input"
          data-test="date-picker-input"
          :class="[
            `${ disabled ? 'cursor-not-allowed' : 'cursor-pointer' }`,
          ]"
          :value="selectedTime.label"
          :placeholder="placeholder"
          :disabled="disabled"
        />

        <button
          v-show="showClearButton"
          type="button"
          :class="['clear-input-button',
                   {
                     'clear-input-button--hover': !disabled && selectedTime.label,
                   }]"
          @click.stop="clearSelectedTime"
        >
          <XmarkIcon />
        </button>
      </button>

      <!-- calendar -->
      <div
        v-if="isCalendarVisible"
        class="calendar-container"
        :style="{ 'z-index': zIndex }"
      >
        <div class="calendar-header">
          <div class="prev-controller">
            <button
              v-if="isYearCalendarVisible && canGoLastDecade"
              type="button"
              class="controller-button"
              data-test="last-decade"
              @click="goToLastDecade"
            >
              <AnglesLeftIcon />
            </button>

            <button
              v-if="!isYearCalendarVisible && canGoLastYear"
              type="button"
              class="controller-button"
              data-test="last-year"
              @click="goToLastYear"
            >
              <AnglesLeftIcon />
            </button>

            <button
              v-show="isDateCalendarVisible && canGoLastMonth"
              type="button"
              class="controller-button"
              data-test="last-month"
              @click="goToLastMonth"
            >
              <AngleLeftIcon />
            </button>
          </div>

          <div class="year-month-controller">
            <button
              type="button"
              class="controller-button"
              data-test="year-button"
              @click="setCalendarVisibility(CalendarType.YEAR)"
            >
              {{ displayYear }}
            </button>

            <button
              v-show="isDateCalendarVisible"
              type="button"
              class="controller-button"
              data-test="month-button"
              @click="setCalendarVisibility(CalendarType.MONTH)"
            >
              {{ getCalendarLang(lang).month[monthOnCalendar] }}
            </button>
          </div>

          <div class="next-controller">
            <button
              v-show="isDateCalendarVisible"
              type="button"
              class="controller-button"
              data-test="next-month"
              @click="goToNextMonth"
            >
              <AngleRightIcon />
            </button>

            <button
              v-if="isYearCalendarVisible"
              type="button"
              class="controller-button"
              data-test="next-decade"
              @click="goToNextDecade"
            >
              <AnglesRightIcon />
            </button>

            <button
              v-else
              type="button"
              class="controller-button"
              data-test="next-year"
              @click="goToNextYear"
            >
              <AnglesRightIcon />
            </button>
          </div>
        </div>

        <div v-if="isYearCalendarVisible">
          <YearCalendar
            :type="type"
            :lang="lang"
            :calendar-year-type="yearType"
            :default-full-date="selectedTime"
            :calendar-year="yearOnCalendar"
            :decade-range="decadeRange"
            @click="handleYearChange"
          />
        </div>

        <div v-if="isMonthCalendarVisible">
          <MonthCalendar
            :type="type"
            :lang="lang"
            :calendar-year-type="yearType"
            :calendar-year="yearOnCalendar"
            :default-full-date="selectedTime"
            @click="handleMonthChange"
          />
        </div>

        <div v-if="isDateCalendarVisible">
          <DateCalendar
            :lang="lang"
            :calendar-year-type="yearType"
            :calendar-year="yearOnCalendar"
            :calendar-month="monthOnCalendar"
            :default-full-date="selectedTime"
            @click="handleDateChange"
          />
        </div>

        <YearTypeSwitch
          :calendar-year-type="yearType"
          :is-visible="isSwitchAllowed"
          :lang="lang"
          @click="handleChangeYearType"
        />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {
  computed, defineComponent, onMounted, ref, toRefs, watch, type PropType
} from 'vue';
import { getCalendarLang, getRepublicEraYear, setDatePickerLabel } from '@/utils';
import {
  Language, YearType, type SelectedTime, CalendarType
} from '@/interfaces';
import DateCalendar from './calendar/DateCalendar.vue';
import MonthCalendar from './calendar/MonthCalendar.vue';
import YearCalendar from './calendar/YearCalendar.vue';
import YearTypeSwitch from './YearTypeSwitch.vue';
import XmarkIcon from './icons/XmarkIcon.vue';
import AngleLeftIcon from './icons/AngleLeftIcon.vue';
import AnglesLeftIcon from './icons/AnglesLeftIcon.vue';
import AngleRightIcon from './icons/AngleRightIcon.vue';
import CalendarDayIcon from './icons/CalendarDayIcon.vue';
import AnglesRightIcon from './icons/AnglesRightIcon.vue';

const DEFAULT_SELECTED_TIME: SelectedTime = {
  label: '',
  timeValue: undefined
};

export default defineComponent({
  components: {
    XmarkIcon,
    AngleLeftIcon,
    AnglesLeftIcon,
    AngleRightIcon,
    CalendarDayIcon,
    AnglesRightIcon,
    DateCalendar,
    YearCalendar,
    MonthCalendar,
    YearTypeSwitch
  },
  props: {
    modelValue: {
      type: Date || String,
      default: ''
    },
    lang: {
      type: String as PropType<Language>,
      default: Language.ZH_TW
    },
    calendarYearType: {
      type: String as PropType<YearType>,
      default: YearType.RepublicEra
    },
    placeholder: {
      type: String,
      default: ''
    },
    type: {
      type: String as PropType<CalendarType>,
      default: CalendarType.DATE
    },
    defaultValue: {
      type: String,
      default: ''
    },
    disabled: {
      type: Boolean,
      default: false
    },
    zIndex: {
      type: Number,
      default: 1
    },
    showClearButton: {
      type: Boolean,
      default: true
    }
  },
  emits: ['update:modelValue'],
  setup(props, { emit }) {
    const {
      type, defaultValue, disabled, calendarYearType, lang, modelValue
    } = toRefs(props);
    const isCalendarVisible = ref(false);
    const isDateCalendarVisible = ref(false);
    const isMonthCalendarVisible = ref(false);
    const isYearCalendarVisible = ref(false);

    const currentMonth = new Date().getMonth();
    const currentYear = new Date().getFullYear();

    const decadeRange = ref<number[]>([]);
    const yearOnCalendar = ref(currentYear);
    const monthOnCalendar = ref(currentMonth);

    const yearType = ref<YearType>(calendarYearType.value);
    const canGoLastYear = ref(true);
    const canGoLastMonth = ref(true);
    const canGoLastDecade = ref(true);
    const selectedTime = ref({ ...DEFAULT_SELECTED_TIME });

    const displayYear = computed(() => {
      const startYear = decadeRange?.value[0];
      const endYear = decadeRange?.value[1];
      const startYearLabel = lang.value === 'zhTW' ? '民國 ' : 'ROC ';
      const endYearLabel = lang.value === 'zhTW' ? ' 年' : '';

      if (yearType.value === YearType.CommonEra) {
        return isYearCalendarVisible.value ? `${startYear} - ${endYear}` : yearOnCalendar.value;
      }

      return isYearCalendarVisible.value
        ? `${startYearLabel}${Math.max(getRepublicEraYear(startYear), 1)} - ${getRepublicEraYear(endYear)}${endYearLabel}`
        : `${startYearLabel}${getRepublicEraYear(yearOnCalendar.value)}${endYearLabel}`;
    });

    const isSwitchAllowed = computed(() => {
      // if currently selecting year, return the year on calendar is larger than the decade
      if (isYearCalendarVisible.value) {
        const beginDecadeYearOnCalendar = yearOnCalendar.value - (yearOnCalendar.value % 10);
        return beginDecadeYearOnCalendar >= 1910; // 1912 - (1912 % 10)
      }
      return getRepublicEraYear(yearOnCalendar.value) > 0;
    });

    const getDecadeRange = () => {
      const decadeStartYear: number = Math.floor(yearOnCalendar.value / 10) * 10;
      decadeRange.value = [decadeStartYear, decadeStartYear + 9];

      const republicEraYear = getRepublicEraYear(decadeRange.value[0]);
      const isRepublicEra = yearType.value === YearType.RepublicEra;
      canGoLastDecade.value = isRepublicEra
        ? republicEraYear >= 1
        : decadeRange.value[0] > 100;
    };

    const setCalendarVisibility = (calendarType: CalendarType) => {
      getDecadeRange();
      const visibilityMap = {
        [CalendarType.DATE]: [true, false, false],
        [CalendarType.MONTH]: [false, true, false],
        [CalendarType.YEAR]: [false, false, true]
      };

      [isDateCalendarVisible.value,
        isMonthCalendarVisible.value,
        isYearCalendarVisible.value] = visibilityMap[calendarType];
    };

    const initCalendar = () => {
      setCalendarVisibility(type.value);

      if (defaultValue.value || modelValue.value) {
        const defaultTime = new Date(defaultValue.value || modelValue.value);
        const defaultYear = defaultTime.getFullYear();

        if ((yearType.value === YearType.RepublicEra) && (defaultYear <= 1911)) {
          yearType.value = YearType.CommonEra;
        } else {
          yearType.value = calendarYearType.value;
        }

        selectedTime.value.label = setDatePickerLabel({
          calendarYearType: yearType.value,
          selectedDate: defaultTime,
          formatYear: defaultYear,
          datePickerType: type.value
        });

        selectedTime.value.timeValue = defaultTime;

        monthOnCalendar.value = new Date(defaultTime).getMonth();
        yearOnCalendar.value = new Date(defaultTime).getFullYear();
      }
    };

    const toggleCalender = () => {
      if (!disabled.value) isCalendarVisible.value = !isCalendarVisible.value;
    };

    const goToNextMonth = () => {
      yearOnCalendar.value = (monthOnCalendar.value === 11) ? yearOnCalendar.value + 1 : yearOnCalendar.value;
      monthOnCalendar.value = (monthOnCalendar.value + 1) % 12;
    };

    const goToLastMonth = () => {
      yearOnCalendar.value = (monthOnCalendar.value === 0) ? yearOnCalendar.value - 1 : yearOnCalendar.value;
      monthOnCalendar.value = (monthOnCalendar.value === 0) ? 11 : monthOnCalendar.value - 1;
    };

    const goToYear = (year: number) => {
      yearOnCalendar.value = year;
      getDecadeRange();
    };

    const goToNextYear = () => {
      goToYear(yearOnCalendar.value + 1);
    };

    const goToLastYear = () => {
      goToYear(yearOnCalendar.value - 1);
    };

    const goToNextDecade = () => {
      goToYear(yearOnCalendar.value + 10);
    };

    const goToLastDecade = () => {
      goToYear(yearOnCalendar.value - 10);
    };

    const handleDateChange = (time: SelectedTime) => {
      selectedTime.value = time;
      isCalendarVisible.value = false;
    };

    const handleMonthChange = (time: SelectedTime) => {
      selectedTime.value = time;
      monthOnCalendar.value = new Date(time.timeValue as Date).getMonth();

      if (type.value === CalendarType.DATE) {
        setCalendarVisibility(CalendarType.DATE);
        return;
      }

      isCalendarVisible.value = false;
    };

    const handleYearChange = (time: SelectedTime) => {
      selectedTime.value = time;

      if (type.value !== CalendarType.YEAR) {
        setCalendarVisibility(CalendarType.MONTH);
        yearOnCalendar.value = new Date(time.timeValue as Date).getFullYear();
        return;
      }

      isCalendarVisible.value = false;
    };

    const handleChangeYearType = (selectedYearType: YearType) => {
      yearType.value = selectedYearType;
      getDecadeRange();
    };

    const clearSelectedTime = () => {
      selectedTime.value = { ...DEFAULT_SELECTED_TIME };
      isCalendarVisible.value = false;

      emit('update:modelValue', selectedTime.value.timeValue);
    };

    onMounted(() => {
      initCalendar();
    });

    watch([yearOnCalendar, yearType], () => {
      canGoLastYear.value = !((yearType.value === YearType.RepublicEra
        && yearOnCalendar.value <= 1912)
        || (decadeRange?.value[0] <= 100)
      );
    }, { immediate: true });

    watch([yearOnCalendar, yearType, monthOnCalendar], () => {
      canGoLastMonth.value = !(yearType.value === YearType.RepublicEra
      && yearOnCalendar.value <= 1912
      && monthOnCalendar.value === 0);
    }, { immediate: true });

    watch(selectedTime, () => {
      if (!isCalendarVisible.value && selectedTime.value.timeValue) {
        emit('update:modelValue', selectedTime.value.timeValue);
      }
    }, { deep: true });

    return {
      yearType,
      displayYear,
      decadeRange,
      selectedTime,
      canGoLastYear,
      yearOnCalendar,
      canGoLastMonth,
      canGoLastDecade,
      monthOnCalendar,
      isSwitchAllowed,
      isCalendarVisible,
      isDateCalendarVisible,
      isMonthCalendarVisible,
      isYearCalendarVisible,
      CalendarType,
      goToNextYear,
      goToLastYear,
      goToNextMonth,
      goToLastMonth,
      toggleCalender,
      goToNextDecade,
      goToLastDecade,
      getCalendarLang,
      handleDateChange,
      handleYearChange,
      clearSelectedTime,
      handleMonthChange,
      getRepublicEraYear,
      handleChangeYearType,
      setCalendarVisibility
    };
  }
});
</script>

<style scoped lang="scss">
button {
  background: transparent;
  border-style: none;
}

.date-picker-container {
  position: relative;
  width: 100%;
  height: 100%;
}

.input-container {
  width: 100%;
  height: 100%;
  padding: 4px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  background: #ffffff;
  border: 1px solid #cbd2d5;

  .input-icon {
    padding: 0px 4px;
  }

  .date-picker-input {
    width: 100%;
    height: 100%;
    padding: 2px;
    background:#ffffff;
    border-style: none;

    &:focus {
      outline-style: none;
    }
  }

  .clear-input-button {
    display: none;
    position: absolute;
    cursor: pointer;
    right: 4px;
    top: 0;
    bottom: 0;
  }

  &:hover {
    .clear-input-button--hover {
      display: flex;
      align-items: center;
    }
  }
}

.calendar-container {
  position: absolute;
  border-radius: 4px;
  top: 120%;
  box-shadow: rgba(0, 0, 0, 0.05) 0px 6px 24px 0px, rgba(0, 0, 0, 0.08) 0px 0px 0px 1px;
  background: #ffffff;
  padding: 8px;
  color: #6a6c6d;
  width: 300px;
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 2px;

  .prev-controller, .next-controller {
    display: flex;
    align-items: center;
    height: 100%;
  }

  .year-month-controller {
    display: flex;
    flex-grow: 1;
    justify-content: center;
    transform: translate(0, -1px);

    .controller-button {
      font-size: 16px;
      &:hover {
        color: #4390BC;
      }
    }
  }

  .controller-button {
    display: flex;
    align-items: center;
    cursor: pointer;
    padding: 0 4px;
  }
}
</style>
