# timetable-calculator
A library that helps you to calculate your time slots for rendering timesheet or timetable.

## Getting started
  1. Install this library

  ```
  npm i --save timetable-calculator
  ```

  2. import function from this library
  ```js
  import { generateTimetable } from 'timetable-calculator'
  ```

## API
`generateTimetable` is function that calculate time slot.

```js
// main.js
export { generateTimetable } from './timetable-generator'
```

## Example
`generateTimetable(timeslot)` receive timeslot as input that is `array` with key `name`, `start`, `duration` and `row`.
`generateTimetable` will return `overlap` in row and sort by start and row.

```js
// timetable-generator.test.js
import { expect } from 'chai'
import { generateTimetable } from './timetable-generator'

describe('generateTimetable', () => {
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
      { name: 'Algorithm', start: 0, duration: 3, row: 0, overlap: 0 },
      { name: 'TDD', start: 2, duration: 4, row: 0, overlap: 1 },
      { name: 'Javascript', start: 1, duration: 3, row: 1, overlap: 0 },
      { name: 'CI', start: 2, duration: 2, row: 1, overlap: 1 },
      { name: 'Agile & Scrum', start: 3, duration: 3, row: 1, overlap: 2 },
      { name: 'Docker', start: 1, duration: 4, row: 2, overlap: 0 },
      { name: 'Web Development', start: 5, duration: 3, row: 2, overlap: 0 }
    ])
  })
})
```

## Implementation
`generateTimetable` is a function that contains
- Slot preparation (`createSortedChunkRow, getSizeAllSlots, sortSlotByStart, createFreeTimeslot`)
- Handle overlapping (`findOverlapIndex`, `updateCountOverlapSlot`)

```js
// timetable-generator.js
import _ from 'lodash'
import { createSortedChunkRow, getSizeAllSlots, sortSlotByStart, createFreeTimeslot } from './slot-preparation'
import { findOverlapIndex, updateCountOverlapSlot } from './overlap-slot'

export const generateTimetable = (slots) => {
  let timetable = []
  const chunks = createSortedChunkRow(slots)
  const fullsize = getSizeAllSlots(slots)
  _.forEach(chunks, (chunk) => { // Todo map or foreach
    const sortedChunk = sortSlotByStart(chunk)
    let countOverlapSlot = createFreeTimeslot(fullsize)
    _.forEach(sortedChunk, (newSlot) => {
      const overlap = findOverlapIndex(countOverlapSlot, newSlot)
      timetable.push({ ...newSlot, overlap })
      countOverlapSlot = updateCountOverlapSlot(countOverlapSlot, newSlot)
    })
  })
  return timetable
}
```

### Prepare Slot
There are mini functions that can prepare slot before integration.

```js
// slot-preparation.js
import _ from 'lodash'

export const createFreeTimeslot = (size) => _.range(0, size, 0)
export const isInTimeRange = (timeTarget, start, end) => _.inRange(timeTarget, start, end + 1)
export const sortSlotByStart = (slots) => _.sortBy(slots, (slot) => slot.start)
export const getEnd = (start, duration) => start + duration - 1

export const getSizeAllSlots = (slots) => {
  const { start, duration } = _.maxBy(slots, (slot) => {
    const { start, duration } = slot
    return getEnd(start, duration) + 1
  })
  return getEnd(start, duration) + 1
}

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

export const convertSlotToArray = (slot) => {
  const { start, duration } = slot
  const end = getEnd(start, duration)
  const size = end + 1
  return _.range(size).map((value, index) => isInTimeRange(index, start, end) ? 1 : 0)
}
```

### Testing
There are some examples about simple results of them.

```js
// slot-preparation.test.js
import { expect } from 'chai'
import {
  createFreeTimeslot,
  isInTimeRange,
  getEnd,
  getSizeAllSlots,
  sortSlotByStart,
  createSortedChunkRow,
  convertSlotToArray
} from './slot-preparation'

describe('slot-preparation', () => {
  it('should create free timeslot with input size', () => {
    expect(createFreeTimeslot(3)).to.eql([0, 0, 0])
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

  it('should calculate correct end slot when pass start and duration', () => {
    const start = 3
    const duration = 5
    expect(getEnd(start, duration)).to.equal(7)
  })

  it('should get size all slots', () => {
    const timeslot = [{ start: 0, duration: 3 }, { start: 1, duration: 4 }]
    expect(getSizeAllSlots(timeslot)).to.equal(5)
  })

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

  it('should convert slot to array', () => {
    expect(convertSlotToArray({ start: 1, duration: 3 })).to.eql([0, 1, 1, 1])
  })
})
```

### Handle overlap slot
`updateCountOverlapSlot` When it receives new slot, it will sum all slot each columns.
`findOverlapIndex` When slots overlap in the same row, it will return a key (`overlap`) as mini row.

```js
// overlap-slot.js
import _ from 'lodash'
import { convertSlotToArray, getEnd, isInTimeRange } from './slot-preparation'

export const updateCountOverlapSlot = (countOverlapSlot, newSlot) => {
  const newSlotArray = convertSlotToArray(newSlot)
  return _.map(countOverlapSlot, (sumColumnSlot, index) => (
    sumColumnSlot + (newSlotArray[index] || 0)
  ))
}

export const findOverlapIndex = (countOverlapSlot, newSlot) => {
  const size = countOverlapSlot.length
  const { start, duration } = newSlot
  const end = getEnd(start, duration)
  const overlapTargetSlot = _.map(countOverlapSlot, (sumColumn, index) => {
    return isInTimeRange(index, start, end) ? sumColumn - 1 : 0
  })
  const newSlotArray = convertSlotToArray(newSlot)
  const overlapIndex = _.max(
    _.map(_.range(size), (index) => {
      return overlapTargetSlot[index] + (newSlotArray[index] || 0)
    })
  )
  return overlapIndex
}
```

### Testing
There are some examples how they can be used.

```js
// overlap-slot.test.js
import { expect } from 'chai'
import { findOverlapIndex, updateCountOverlapSlot } from './overlap-slot'

describe('overlap-slot', () => {
  it('should find index of overlap slots when it overlap', () => {
    const allSlot = [1, 0, 0, 0, 0];
    const newSlot = { start: 0, duration: 2 }
    expect(findOverlapIndex(allSlot, newSlot)).to.equal(1)
  })

  it('should find index of overlap slots when it doesn\'t overlap', () => {
    const allSlot = [0, 0, 0, 0];
    const newSlot = { start: 1, duration: 2 }
    expect(findOverlapIndex(allSlot, newSlot)).to.equal(0)
  })

  it('should update count overlap slot when it has new slot', () => {
    const countOverlapSlot = [0, 1, 2, 1, 0]
    const newSlot = { start: 2, duration: 2 }
    expect(updateCountOverlapSlot(countOverlapSlot, newSlot)).to.eql([0, 1, 3, 2, 0])
  })
})
```
