<template>
  <div class="calendar-multi-day-component column fit no-wrap">
    <!-- week nav -->
    <template v-if="numDays === 1">
      <calendar-header-nav
        time-period-unit="days"
        :time-period-amount="navDays"
        :move-time-period-emit="eventRef + ':navMovePeriod'"
        :calendar-locale="calendarLocale"
      >
        {{ formatDate(workingDate, 'EEEE, MMMM d, yyyy')}}
      </calendar-header-nav>
    </template>
    <template v-else>
      <calendar-header-nav
        time-period-unit="days"
        :time-period-amount="navDays"
        :move-time-period-emit="eventRef + ':navMovePeriod'"
      >
        {{ getHeaderLabel() }}
      </calendar-header-nav>
    </template>

    <div v-if="numDays > 1" class="calendar-time-margin">
      <calendar-day-labels
        :number-of-days="numDays"
        :show-dates="true"
        :start-date="workingDate"
        :force-start-of-week="forceStartOfWeek"
        :full-component-ref="fullComponentRef"
        :sunday-first-day-of-week="sundayFirstDayOfWeek"
        :calendar-locale="calendarLocale"
      />
    </div>

    <!-- all day events -->
    <div class="calendar-time-margin">
      <calendar-all-day-events
        :number-of-days="numDays"
        :start-date="weekDateArray[0]"
        :parsed="parsed"
        :event-ref="eventRef"
        :prevent-event-detail="preventEventDetail"
        :calendar-locale="calendarLocale"
        :calendar-timezone="calendarTimezone"
      />
    </div>

    <!-- content -->
    <q-scroll-area
      :style="getScrollStyle"
      :class="getScrollClass"
    >
      <div class="col">
        <div class="calendar-day row">
          <calendar-time-label-column
            :calendar-locale="calendarLocale"
          />
          <div class="calendar-multiple-days col row">
            <calendar-day-column
              v-for="thisDate in weekDateArray"
              :key="makeDT(thisDate).toISODate()"
              :start-date="thisDate"
              :date-events="dateGetEvents(thisDate, true)"
              column-css-class="calendar-day-column-content"
              :style="{ 'width': dayCellWidth }"
              :event-ref="eventRef"
              :prevent-event-detail="preventEventDetail"
              :calendar-locale="calendarLocale"
              :calendar-timezone="calendarTimezone"
              :allow-editing="allowEditing"
            />
          </div>
        </div>
      </div>

    </q-scroll-area>

    <calendar-event-detail
      ref="defaultEventDetail"
      v-if="!preventEventDetail"
      :event-object="eventDetailEventObject"
      :event-ref="eventRef"
      :calendar-locale="calendarLocale"
      :calendar-timezone="calendarTimezone"
      :allow-editing="allowEditing"
    />

  </div>
</template>

<script>
  import CalendarMixin from './mixins/CalendarMixin'
  import CalendarEventMixin from './mixins/CalendarEventMixin'
  import CalendarParentComponentMixin from './mixins/CalendarParentComponentMixin'
  import CalendarEvent from './CalendarEvent'
  import CalendarDayColumn from './CalendarDayColumn'
  import CalendarTimeLabelColumn from './CalendarTimeLabelColumn'
  import CalendarDayLabels from './CalendarDayLabels'
  import CalendarHeaderNav from './CalendarHeaderNav'
  import CalendarAllDayEvents from './CalendarAllDayEvents'
  import CalendarEventDetail from './CalendarEventDetail'
  import {
    QBtn,
    QTooltip,
    QScrollArea,
    scroll
  } from 'quasar'
  const { getScrollTarget, setScrollPosition } = scroll
  export default {
    name: 'CalendarMultiDay',
    mixins: [CalendarParentComponentMixin, CalendarMixin, CalendarEventMixin],
    props: {
      numDays: {
        type: Number,
        default: 7
      },
      navDays: {
        type: Number,
        default: 7
      },
      forceStartOfWeek: {
        type: Boolean,
        default: true
      },
      dayCellHeight: {
        type: Number,
        default: 5
      },
      dayCellHeightUnit: {
        type: String,
        default: 'rem'
      },
      scrollStyle: {
        type: Object,
        default: function () {
          return {}
        }
      },
      scrollHeight: {
        type: String,
        default: 'auto'
      },
      fullComponentRef: String
    },
    components: {
      CalendarEvent,
      CalendarDayColumn,
      CalendarTimeLabelColumn,
      CalendarDayLabels,
      CalendarHeaderNav,
      CalendarAllDayEvents,
      CalendarEventDetail,
      QBtn,
      QTooltip,
      QScrollArea
    },
    data () {
      return {
        workingDate: new Date(),
        weekDateArray: [],
        dayRowArray: [],
        parsed: this.getDefaultParsed(),
        thisNavRef: this.createNewNavEventName(),
        eventDetailEventObject: {}
      }
    },
    computed: {
      dayCellWidth: function () {
        return this.calculateDayCellWidth(this.numDays)
      },
      getScrollStyle: function () {
        if (this.scrollStyle.length > 0) {
          return this.scrollStyle
        }
        else {
          return {
            'height': this.scrollHeight
          }
        }
      },
      getScrollClass: function () {
        if (this.scrollHeight === 'auto') {
          return {
            'col': true
          }
        }
        else {
          return {}
        }
      }
    },
    methods: {
      getHeaderLabel: function () {
        if (this.forceStartOfWeek) {
          let dateReturn = ''
          let bookendDates = this.getForcedWeekBookendDates()
          if (bookendDates.first.month !== bookendDates.last.month) {
            dateReturn += bookendDates.first.toFormat('MMM')
            if (bookendDates.first.year !== bookendDates.last.year) {
              dateReturn += bookendDates.first.toFormat(' yyyy')
            }
            dateReturn += ' - '
          }
          dateReturn += bookendDates.last.toFormat('MMM yyyy')
          return dateReturn
        }
        else {
          return this.makeDT(this.workingDate).toFormat('MMMM yyyy')
        }
      },
      doUpdate: function () {
        this.mountSetDate()
        this.buildWeekDateArray(this.numDays, this.sundayFirstDayOfWeek)
        this.$nextTick(() => {
          this.scrollToFirstDay()
        })
      },
      handleNavMove: function (unitType, amount) {
        this.moveTimePeriod(unitType, amount)
        this.$emit(
          this.eventRef + ':navMovePeriod',
          {
            unitType: unitType,
            amount: amount
          }
        )
        this.buildWeekDateArray()
      },
      scrollToElement: function (el) {
        let target = getScrollTarget(el)
        let offset = el.offsetTop - el.scrollHeight
        let duration = 0
        setScrollPosition(target, offset, duration)
      },
      scrollToFirstDay: function () {
        let thisId = this.getDayHourId(
          this.eventRef,
          this.weekDateArray[0],
          (this.dayDisplayStartHour + 1)
        )
        let thisEl = document.getElementById(thisId)
        this.scrollToElement(thisEl)
      }
    },
    mounted: function () {
      this.doUpdate()
      this.handlePassedInEvents()
      this.$root.$on(
        this.eventRef + ':navMovePeriod',
        this.handleNavMove
      )
      this.$root.$on(
        'click-event-' + this.eventRef,
        this.handleEventDetailEvent
      )
      this.$root.$on(
        'update-event-' + this.eventRef,
        this.handleEventUpdate
      )
    },
    watch: {
      startDate: 'handleStartChange',
      eventArray: 'getPassedInEventArray',
      parsedEvents: 'getPassedInParsedEvents'
    }
  }
</script>

<style lang="stylus">
  @import 'calendar.vars.styl'

  .calendar-multi-day-component
    .calendar-time-margin
      margin-left $dayTimeLabelWidth
    .calendar-header
      .calendar-header-label
        font-size 1.25em
        font-weight bold
    .calendar-day
      .calendar-day-column-label
        width $dayTimeLabelWidth
      .calendar-day-column-content
        border-right $borderThin
        position relative
      .calendar-day-time
        padding-right .5em
        border-right $borderOuter
      .calendar-day-time-content
        border-top $borderThin

</style>
