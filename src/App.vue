<script setup lang="ts">
import { reactive, ref } from 'vue';
import { useI18n } from 'vue-i18n';

const { t } =useI18n();

interface Turn {
  keySlots: KeySlotState[];
  colorSlots: (ColorSlotEmpty | ColorPeg)[];
  colorSlotsPlaced: number;
}

enum KeySlotState {
  Empty,
  RightColor,
  RightColorAndPosition
}

type ColorSlotEmpty = '';
const ColorSlotEmpty = '';
type ColorPeg = string;

const ColorPegs: ColorPeg[] = [
  // 浅色
  '#EF9A9A',
  '#CE93D8',
  '#90CAF9',
  '#80CBC4',
  '#FFCC80',
  '#BCAAA4',
  '#E6EE9C',
  '#B0BEC5',
  // 深色
  '#B71C1C',
  '#4A148C',
  '#01579B',
  '#004D40',
  '#F57F17',
  '#3E2723',
  '#827717',
  '#263238',
  // 黑白灰
  '#FFFFFF',
  '#BDBDBD',
  '#424242',
  '#000000'
];

enum GameState {
  Ready = 0,
  Running = 1,
  Win = 2,
  Lose = 3
}

class Game {
  public readonly turns = reactive<Turn[]>([]);
  public currTurn: number = 0;
  public colorPegs: ColorPeg[];
  public codePegs: ColorPeg[];
  public gameState = GameState.Ready;

  constructor(private slotNum: number, private colorNum: number, turnNum: number) {
    this.colorPegs = ColorPegs.slice(0, colorNum);

    for (let i = 0; i < turnNum; i++) {
      this.turns.push({
        keySlots: new Array(slotNum).fill(KeySlotState.Empty),
        colorSlots: new Array(slotNum).fill(ColorSlotEmpty),
        colorSlotsPlaced: 0
      })
    }

    const codePegs = new Array<ColorPeg>(slotNum);
    for (let i = 0; i < slotNum; i++) {
      const randomPeg = this.colorPegs[Math.floor(Math.random() * this.colorPegs.length)];
      codePegs[i] = randomPeg;
    }
    this.codePegs = codePegs;
  }

  setColorSlot(slotIndex: number, colorPeg: ColorPeg | ColorSlotEmpty) {
    if (this.gameState === GameState.Ready) {
      this.gameState = GameState.Running;
    }
    if (this.gameState === GameState.Lose || this.gameState === GameState.Win) {
      return;
    }

    const turn = this.turns[this.currTurn];
    const slots = turn.colorSlots;
    if (colorPeg === ColorSlotEmpty && slots[slotIndex] !== ColorSlotEmpty) {
      turn.colorSlotsPlaced--;
    }
    if (colorPeg !== ColorSlotEmpty && slots[slotIndex] === ColorSlotEmpty) {
      turn.colorSlotsPlaced++;
    }
    this.turns[this.currTurn].colorSlots[slotIndex] = colorPeg;
  }

  nextTurn() {
    if (!this.isColorSlotsFull() || this.gameState !== GameState.Running) {
      return;
    }
    if (this.currTurn === this.turns.length) {
      this.gameState = GameState.Lose;
      return;  
    }

    const turn = this.turns[this.currTurn];
    const colorSlots = turn.colorSlots;
    const codeSlots = this.codePegs;
    const keySlots = turn.keySlots;
    const codeColorPegsMap = new Map<ColorPeg, number>();
    const remainingColorPegs: ColorPeg[] = [];
    let keySlotsIndex = 0;

    for (let i = 0; i < colorSlots.length; i++) {
      if (colorSlots[i] === codeSlots[i]) {
        keySlots[keySlotsIndex++] = KeySlotState.RightColorAndPosition;
      } else {
        remainingColorPegs.push(colorSlots[i]);
        if (!codeColorPegsMap.has(codeSlots[i])) {
          codeColorPegsMap.set(codeSlots[i], 0);
        }
        codeColorPegsMap.set(codeSlots[i], codeColorPegsMap.get(codeSlots[i])! + 1);
      }
    }

    for (let i = 0; i < remainingColorPegs.length; i++) {
      const count = codeColorPegsMap.get(remainingColorPegs[i]);
      if (count !== undefined && count > 0) {
        keySlots[keySlotsIndex++] = KeySlotState.RightColor;
        codeColorPegsMap.set(remainingColorPegs[i], count - 1);
      }
    }

    if (this.checkWin() || this.checkLost()) {
      return
    }
    this.currTurn++;
  }

  getTurn() {
    return this.turns[this.currTurn];
  }

  private checkWin() {
    for (const slot of this.turns[this.currTurn].keySlots) {
      if (slot !== KeySlotState.RightColorAndPosition) {
        return false;
      }
    }

    this.gameState = GameState.Win;
    return true;
  }

  private checkLost() {
    if (this.currTurn === this.turns.length - 1) {
      this.gameState = GameState.Lose;
      return true; 
    }
    return false;
  }

  getScore() {
    if (this.gameState !== GameState.Win) {
      return 0;
    }
    const numberOfGuess = this.currTurn + 1;
    const expectedNumber = this.slotNum * Math.log2(this.colorNum);
    const perfectNumber = expectedNumber / 2;

    if (numberOfGuess <= perfectNumber) {
      return 3;
    } else if (numberOfGuess <= expectedNumber) {
      return 2;
    }

    return 1;
  }

  isColorSlotsFull() {
    const turn = this.turns[this.currTurn];
    return turn.colorSlotsPlaced === turn.colorSlots.length
  }
}

const game = ref<Game>();
const gameOptions = reactive({
  slotNum: 4,
  colorNum: 6,
  turnNum: 12
});
const settingVisible = ref(false);

function startNewGame() {
  game.value = new Game(gameOptions.slotNum, gameOptions.colorNum, gameOptions.turnNum);
}

startNewGame();

function onColorSlotClick(game: Game, turnIndex: number, slotIndex: number, colorPeg: ColorPeg) {
  if (game.currTurn === turnIndex && !isGameFinished(game)) {
    game.setColorSlot(slotIndex, ColorSlotEmpty);
  } else {
    placedColorPeg(colorPeg);
  }
}

function placedColorPeg(colorPeg: ColorPeg) {
  const turn = game.value!.getTurn();

  const colorSlots = turn.colorSlots;
  for (let i = 0; i < colorSlots.length; i++) {
    if (colorSlots[i] === ColorSlotEmpty) {
      game.value!.setColorSlot(i, colorPeg);
      break;
    }
  }
}

function getKeySlotColor (slot: KeySlotState) {
  switch (slot) {
    case KeySlotState.RightColorAndPosition:
      return '#636363';
    case KeySlotState.RightColor:
      return '#ffffff';
    default:
      return '';
  }
}

function isGameFinished(game: Game) {
  return game.gameState === GameState.Win || game.gameState === GameState.Lose;
}

function navToHowToPlay() {
  const url = t('urls.howToPlay');
  window.open(url, '_blank');
}

function onRestart(game: Game) {
  if (game.gameState === GameState.Running) {
    if (window.confirm(t('tips.restartConfirm'))) {
      startNewGame();
    }
  } else {
    startNewGame();
  }
}

function enforceMinMax(e: Event) {
  const el = e.target as HTMLInputElement;

  if (el.value != "") {
    if (parseInt(el.value) < parseInt(el.min)) {
      el.value = el.min;
    }
    if (parseInt(el.value) > parseInt(el.max)) {
      el.value = el.max;
    }
  }
}

</script>

<template>
  <div class="app">
    <header class="header"></header>
    <main>
      <div v-if="game" class="board">
        <div class="playable-area">
          <div class="decoding-board">
            <div v-for="(turn, turnIndex) in game.turns" class="decoding-row" :class="{ 'is-active': game.currTurn === turnIndex && !isGameFinished(game) }">
              <div class="key-slots">
                <div class="key-slot-row">
                  <div v-for="(keySlot) in turn.keySlots.slice(0, turn.colorSlots.length / 2)" class="key-slot">
                    <div class="key-peg" :style="{ backgroundColor: getKeySlotColor(keySlot) }"></div>
                  </div>
                </div>
                <div class="key-slot-row">
                  <div v-for="(keySlot) in turn.keySlots.slice(turn.colorSlots.length / 2)" class="key-slot">
                    <div class="key-peg" :style="{ backgroundColor: getKeySlotColor(keySlot) }"></div>
                  </div>
                </div>
              </div>
              <div class="color-slot-row">
                <div v-for="(colorSlot, index) in turn.colorSlots" class="color-slot">
                  <div v-if="colorSlot !== ColorSlotEmpty" class="color-peg" :style="{ backgroundColor: colorSlot }" @click="onColorSlotClick(game, turnIndex, index, colorSlot)"></div>
                </div>
              </div>
              <div v-if="game.currTurn === turnIndex && !isGameFinished(game)" class="next-button" :class="{ 'is-enabled': game.isColorSlotsFull() }" @click="game.nextTurn()">
                <img class="confirm-icon" src="/images/confirm.png" />
              </div>
            </div>
          </div>
          <div class="color-pegs">
            <div v-for="colorPeg in game.colorPegs" class="color-slot" @click="placedColorPeg(colorPeg)">
              <div class="color-peg" :style="{ backgroundColor: colorPeg }"></div>
            </div>
          </div>
        </div>
        <div class="code-board">
          <div v-if="isGameFinished(game)" class="color-slot-row">
            <div v-for="colorPeg in game.codePegs" class="color-slot" :style="{ backgroundColor: colorPeg }">
              <div class="color-peg" :style="{ backgroundColor: colorPeg }"></div>
            </div>
          </div>
          <div v-else class="shield">
            Mastermind
          </div>
          <div v-if="game.gameState === GameState.Lose" class="fail-board">
            <img class="lost-icon" src="/images/fail.png" />
            Game Over
          </div>
          <div v-if="game.gameState === GameState.Win" class="win-board">
            <div class="score-board">
              <img v-for="n in game.getScore()" class="win-icon" src="/images/star.png" />
            </div>
            You Win
          </div>
        </div>
        <div v-if="settingVisible" class="setting">
          <div class="setting-item">
            <label>{{ t('labels.language') }}</label>
            <select v-model="$i18n.locale">
              <option v-for="locale in $i18n.availableLocales" :key="`locale-${locale}`" :value="locale">{{ locale }}</option>
            </select>
          </div>
          <div class="setting-item">
            <label>{{ t('labels.colorNum') }}</label>
            <input v-model="gameOptions.colorNum" type="number" min="2" :max="ColorPegs.length" @input="enforceMinMax" />
          </div>
          <div class="setting-item">
            <label>{{ t('labels.slotNum') }}</label>
            <input v-model="gameOptions.slotNum" type="number" min="2" @input="enforceMinMax" />
          </div>
          <div class="setting-item">
            <label>{{ t('labels.turnNum') }}</label>
            <input v-model="gameOptions.turnNum" type="number" min="1" @input="enforceMinMax" />
          </div>
        </div>
        <div class="actions">
          <div class="action question-action" :title="t('tips.question')" @click="navToHowToPlay">
            <img class="action-icon" src="/images/question-mark.png" />
          </div>
          <div class="action setting-action" :title="t('tips.setting')" :class="{ 'is-active': settingVisible }" @click="settingVisible = !settingVisible">
            <img class="action-icon" src="/images/setting.png" />
          </div>
          <div class="action restart-action" :title="t('tips.restart')" @click="onRestart(game)">
            <img class="action-icon" src="/images/restart.png" />
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style scoped>
.board {
  --mg-boder: 1px solid #e5e5e5;
  background-color: #f0f0f0;
  border-radius: 4px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.16);
  user-select: none;
  overflow: hidden;
}

.playable-area {
  display: flex;
}

.decoding-board {
  display: flex;
  flex-flow: column;
}

.decoding-row {
  position: relative;
  display: flex;
  border-bottom: var(--mg-boder);
  align-items: center;
  padding: 4px 0;
}

.next-button {
  position: absolute;
  left: -52px;
  width: 52px;
  top: 0;
  bottom: 0;
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: rgba(255,255,255,0.8);
  letter-spacing: 0.1em;
  border-right: var(--mg-boder);
  transition: all 225ms cubic-bezier(0.4,0,0.6,1);
}

.next-button.is-enabled {
  left: 0;
  cursor: pointer;
}

.next-button.is-enabled:hover {
  background-color: #dbeeec;
}

.confirm-icon {
  width: 36px;
}

.decoding-row.is-active {
  background-color: #fafafa;
}

.decoding-row.is-active .color-slot, .decoding-row.is-active .key-slot {
  background-color: #f7f7f7;
}

.decoding-row:last-child {
  border-bottom: none;
}

.key-slot-row {
  display: flex;
  padding: 0 8px;
}

.color-slot-row {
  display: flex;
  padding: 0 32px;
}

.key-slot {
  width: 14px;
  height: 14px;
  border-radius: 7px;
  margin: 2px;
  box-shadow: inset 0 1px 1px 1px rgba(0,0,0,0.1);
  background-color: #eaeaea;
}

.key-peg {
  width: 14px;
  height: 14px;
  border-radius: 7px;
  box-shadow: 0 1px 2px rgba(0,0,0,0.2);
}

.color-slot {
  width: 32px;
  height: 32px;
  border-radius: 16px;
  box-shadow: inset 0 1px 1px 1px rgba(0,0,0,0.1);
  background-color: #eaeaea;
  margin: 4px 2px;
}

.color-pegs {
  border-left: var(--mg-boder);
  padding: 8px;
}

.color-pegs .color-slot {
  margin: 0 0 6px 0;
}

.color-peg {
  width: 32px;
  height: 32px;
  border-radius: 16px;
  cursor: pointer;
  box-shadow: 0 1px 2px rgba(0,0,0,0.2);
}

.color-peg:hover {
  outline: 2px solid #ffffff;
}

.code-board {
  border-top: var(--mg-boder);
  padding: 8px 0;
  display: flex;
  align-items: center;
  flex-flow: column;
}

.shield {
  font-size: 32px;
  text-align: center;
  color: #d5d5d5;
}

.fail-board, .win-board {
  display: flex;
  flex-flow: column;
  align-items: center;
  font-size: 24px;
  font-weight: bold;
  padding: 16px 0;
}

.fail-board {
  color: #ef9a9a;
}

.win-board {
  color: #ffcc80;
}

.actions {
  display: flex;
  border-top: var(--mg-boder);
}

.action {
  width: 64px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.action:hover {
  background-color: #dbeeec;
}

.action-icon {
  width: 36px;
}

.question-action {
  border-right: var(--mg-boder);
}

.setting-action {
  margin-left: auto;
  border-left: var(--mg-boder);
  border-right: var(--mg-boder);
}

.setting-action.is-active {
  background-color: #dbeeec;
}

.setting {
  display: flex;
  justify-content: center;
}

.setting input, .setting select {
  position: relative;
  background-color: #eaeaea;
  border: var(--mg-boder);
  padding: 4px 6px;
  box-shadow: inset 0 1px 1px 1px rgba(0,0,0,0.1);
  border-radius: 4px;
  color: #606060;
}

.setting input {
  width: 3em;
}

.setting select {
  width: 8em;
}

.setting-item {
  color: #606060;
  font-size: 13px;
  text-align: center;
  display: flex;
  flex-flow: column;
  padding: 4px;
}
</style>
