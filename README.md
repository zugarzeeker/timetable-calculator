# timetable-calculator
A library that helps you to calculate your time slots for rendering timesheet.

```js
// main.js
console.log('Hello essay')
```

```js
// timetable-calculator.js
import _ from 'lodash'

export const getSizeAllSlots = (slots) => {
  const { start, duration } = _.maxBy(slots, (slot) => {
    const { start, duration } = slot
    return getEnd(start, duration) + 1
  })
  return getEnd(start, duration) + 1
}
export const createFreeTimeslot = (size) => _.range(0, size, 0)
export const isInTimeRange = (timeTarget, start, end) => _.inRange(timeTarget, start, end + 1)
export const sortSlotByStart = (slots) => _.sortBy(slots, (slot) => slot.start)
export const getEnd = (start, duration) => start + duration - 1
export const createSortedChunkRow = (slots) => {
  let chunks = []
  _.map(slots, (slot) => {
    const index = slot.row
    if (chunks[index]) {
      chunks[index].push(slot)
    }
    else {
      chunks[index] = [slot]
    }
  })
  return chunks
}

export const updateCountCollapseSlot = (countCollapseSlot, newSlot) => {
  const newSlotArray = convertSlotToArray(newSlot)
  return _.map(countCollapseSlot, (sumColumnSlot, index) => (
    sumColumnSlot + (newSlotArray[index] || 0)
  ))
}

export const convertSlotToArray = (slot) => {
  const { start, duration } = slot
  const end = getEnd(start, duration)
  const size = end + 1
  return _.range(size).map((value, index) => isInTimeRange(index, start, end) ? 1 : 0)
}

export const findCollapseIndex = (countCollapseSlot, newSlot) => {
  const size = countCollapseSlot.length
  const { start, duration } = newSlot
  const end = getEnd(start, duration)
  const collapseTargetSlot = _.map(countCollapseSlot, (sumColumn, index) => {
    return isInTimeRange(index, start, end) ? sumColumn - 1 : 0
  })
  const newSlotArray = convertSlotToArray(newSlot)
  const collapseIndex = _.max(
    _.map(_.range(size), (index) => {
      return collapseTargetSlot[index] + (newSlotArray[index] || 0)
    })
  )
  return collapseIndex
}

export const generateTimetable = (slots) => {
  let timetable = []
  const chunks = createSortedChunkRow(slots)
  const fullsize = getSizeAllSlots(slots)
  _.forEach(chunks, (chunk) => { // Todo map or foreach
    const sortedChunk = sortSlotByStart(chunk)
    let countCollapseSlot = createFreeTimeslot(fullsize)
    _.forEach(sortedChunk, (newSlot) => {
      const collapse = findCollapseIndex(countCollapseSlot, newSlot)
      timetable.push({ ...newSlot, collapse })
      countCollapseSlot = updateCountCollapseSlot(countCollapseSlot, newSlot)
    })
  })
  return timetable
}
```

```js
// timetable-calculator.test.js
import { expect } from 'chai'
import {
  timetable,
  sortSlotByStart,
  findCollapseIndex,
  createFreeTimeslot,
  isInTimeRange,
  getEnd,
  convertSlotToArray,
  createSortedChunkRow,
  updateCountCollapseSlot,
  getSizeAllSlots,
  generateTimetable,
} from './timetable-calculator'

describe('timetable-calculator', () => {
  it('should sort slots by start', () => {
    const initialTimeslots = [{ start: 2 }, { start: 0 }, { start: 1 }]
    const sortedSlots = [{ start: 0 }, { start: 1 }, { start: 2 }]
    const actual = sortSlotByStart(initialTimeslots)
    expect(sortedSlots).to.eql(actual)
  })

  it('should chunk and sort by row', () => {
    const initialTimeslots = [{ row: 0 }, { row: 2 }, { row: 0 }, { row: 1 }]
    const sortedChunkRow = [[{ row: 0 }, { row: 0 }], [{ row: 1 }], [{ row: 2 }]]
    expect(createSortedChunkRow(initialTimeslots)).to.eql(sortedChunkRow)
  })

  it('should find index of collapse slots when it collapse', () => {
    const allSlot = [1, 0, 0, 0, 0];
    const newSlot = { start: 0, duration: 2 }
    expect(findCollapseIndex(allSlot, newSlot)).to.equal(1)
  })

  it('should find index of collapse slots when it doesn\'t collapse', () => {
    const allSlot = [0, 0, 0, 0];
    const newSlot = { start: 1, duration: 2 }
    expect(findCollapseIndex(allSlot, newSlot)).to.equal(0)
  })

  it('should create free timeslot with input size', () => {
    expect(createFreeTimeslot(3)).to.eql([0, 0, 0])
  })

  it('should convert slot to array', () => {
    expect(convertSlotToArray({ start: 1, duration: 3 })).to.eql([0, 1, 1, 1])
  })

  it('should calculate correct end slot when pass start and duration', () => {
    const start = 3
    const duration = 5
    expect(getEnd(start, duration)).to.equal(7)
  })

  it('should get size all slots', () => {
    const timeslot = [{ start: 0, duration: 3 }, { start: 1, duration: 4 }]
    expect(getSizeAllSlots(timeslot)).to.equal(5)
  })

  it('should update count collapse slot when it has new slot', () => {
    const countCollapseSlot = [0, 1, 2, 1, 0]
    const newSlot = { start: 2, duration: 2 }
    expect(updateCountCollapseSlot(countCollapseSlot, newSlot)).to.eql([0, 1, 3, 2, 0])
  })

  it('should be true because 5 is in range [2, 5]', () => {
    const timeTarget = 5
    const start = 2
    const end = 5
    expect(isInTimeRange(timeTarget, start, end)).to.equal(true)
  })

  it('should be false because 3 is not in range when start with 4 and duration with 2', () => {
    const timeTarget = 3
    const start = 4
    const duration = 2
    expect(isInTimeRange(timeTarget, start, getEnd(start, duration))).to.equal(false)
  })

  it('should generate when pass full timeslot', () => {
    const timeslot = [
      { name: 'Algorithm', start: 0, duration: 3, row: 0 },
      { name: 'TDD', start: 2, duration: 4, row: 0 },
      { name: 'Javascript', start: 1, duration: 3, row: 1 },
      { name: 'Agile & Scrum', start: 3, duration: 3, row: 1 },
      { name: 'Web Development', start: 5, duration: 3, row: 2 },
      { name: 'Docker', start: 1, duration: 4, row: 2 },
      { name: 'CI', start: 2, duration: 2, row: 1 }
    ]
    expect(generateTimetable(timeslot)).to.eql([
      { name: 'Algorithm', start: 0, duration: 3, row: 0, collapse: 0 },
      { name: 'TDD', start: 2, duration: 4, row: 0, collapse: 1 },
      { name: 'Javascript', start: 1, duration: 3, row: 1, collapse: 0 },
      { name: 'CI', start: 2, duration: 2, row: 1, collapse: 1 },
      { name: 'Agile & Scrum', start: 3, duration: 3, row: 1, collapse: 2 },
      { name: 'Docker', start: 1, duration: 4, row: 2, collapse: 0 },
      { name: 'Web Development', start: 5, duration: 3, row: 2, collapse: 0 }
    ])
  })
})
```
