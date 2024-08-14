<script lang="ts">
  import { fly } from "svelte/transition";
  import { circOut } from "svelte/easing";
  import { Window } from "@tauri-apps/api/window";
  import { Maximize, Minimize, Minus, X } from "lucide-svelte";

  const appWindow = new Window("main");

  const operations = ["div", "mul", "add", "sub"] as const;
  const operationsDisplay = ["&divide;", "&times;", "+", "-"] as const;

  let currentOperation = $state<(typeof operations)[number] | null>(null);
  let firstNumber = $state<number | null>(null);
  let secondNumber = $state<number | null>(null);

  let numberNegative = $state(false);
  let usingDecimal = $state(false);

  let zerosAdded = $state(0);

  let currentDisplay = $state("");

  let isMaximized = $state(false);

  function handleMaximize() {
    isMaximized = !isMaximized;
    appWindow.toggleMaximize();
  }

  $effect(() => {
    let currentSign = numberNegative ? "-" : "";
    let decimal = usingDecimal ? "." : "";
    let zeros = "";

    if (zerosAdded > 0) {
      zeros = new Array(zerosAdded)
        .fill(null)
        .map(() => "0")
        .reduce((previous, current) => previous + current);
    }

    if (currentOperation && secondNumber?.toString().includes(".")) {
      decimal = "";
    }

    if (!currentOperation && firstNumber?.toString().includes(".")) {
      decimal = "";
    }

    if (secondNumber || currentOperation) {
      currentDisplay =
        currentSign + (secondNumber || 0).toString() + decimal + zeros;

      return;
    }

    currentDisplay =
      currentSign + (firstNumber || 0).toString() + decimal + zeros;
  });

  function toggleNumberNegative() {
    numberNegative = !numberNegative;
  }

  function toggleUsingDecimal() {
    if (currentOperation && secondNumber?.toString().includes(".")) {
      return;
    }

    if (!currentOperation && firstNumber?.toString().includes(".")) {
      return;
    }

    usingDecimal = !usingDecimal;
  }

  function appendNumber(number: number) {
    if (currentOperation !== null) {
      if (secondNumber && secondNumber.toString().length > 10) {
        return;
      }

      let secondNumberHasDecimal = secondNumber?.toString().includes(".");

      if (usingDecimal || secondNumberHasDecimal) {
        if (number === 0) {
          zerosAdded++;

          return;
        }

        secondNumber =
          (secondNumber || 0) +
          number /
            (secondNumber !== null &&
            secondNumber.toString().split(".").length > 1
              ? 10 **
                (secondNumber.toString().split(".")[1].length + 1 + zerosAdded)
              : 10 ** (zerosAdded + 1));

        zerosAdded = 0;

        return;
      }

      secondNumber = (secondNumber || 0) * 10 + number;

      return;
    }

    if (firstNumber && firstNumber.toString().length > 10) {
      return;
    }

    let firstNumberHasDecimal = firstNumber?.toString().includes(".");

    if (usingDecimal || firstNumberHasDecimal) {
      if (number === 0) {
        zerosAdded++;

        return;
      }

      firstNumber =
        (firstNumber || 0) +
        number /
          (firstNumber !== null && firstNumber.toString().split(".").length > 1
            ? 10 **
              (firstNumber.toString().split(".")[1].length + zerosAdded + 1)
            : 10 ** (zerosAdded + 1));

      zerosAdded = 0;

      return;
    }

    firstNumber = (firstNumber || 0) * 10 + number;
  }

  function clear() {
    if (secondNumber) {
      secondNumber = null;
      clearOperands();

      return;
    }

    if (currentOperation) {
      currentOperation = null;

      return;
    }

    firstNumber = null;
    clearOperands();
  }

  function hardClear() {
    clearOperands();
    secondNumber = null;
    firstNumber = null;
    currentOperation = null;
  }

  function clearOperands() {
    zerosAdded = 0;
    numberNegative = false;
    usingDecimal = false;
  }

  function setCurrentOperation(operation: (typeof operations)[number]) {
    submitCurrentNumber();

    currentOperation = operation;
    clearOperands();
  }

  function submitCurrentNumber() {
    if (!currentOperation) {
      if (numberNegative && firstNumber !== null) {
        firstNumber = -firstNumber;
      }

      return;
    }
  }

  function getResult() {
    if (!currentOperation) {
      return;
    }

    let localResults = 0;

    if (firstNumber === null) {
      firstNumber = 0;
    }

    if (secondNumber === null) {
      secondNumber = firstNumber;
    }

    if (numberNegative) {
      secondNumber = -secondNumber;
    }

    switch (currentOperation) {
      case "div":
        localResults = firstNumber / secondNumber;
        break;
      case "add":
        localResults = firstNumber + secondNumber;
        break;
      case "mul":
        localResults = firstNumber * secondNumber;
        break;
      case "sub":
        localResults = firstNumber - secondNumber;
        break;
    }

    hardClear();
    firstNumber = localResults;
  }

  let showCalculator = $state(false);

  $effect(() => {
    showCalculator = true;
  });
</script>

<header
  data-tauri-drag-region
  class="absolute inset-0 inset-x-0 left-0 right-0 top-0 flex h-[30px] select-none justify-end bg-transparent"
>
  <button
    onclick={() => appWindow.minimize()}
    class="inline-flex h-[30px] w-[30px] items-center justify-center px-2 py-1 transition-colors duration-100 hover:bg-zinc-600/50"
  >
    <Minus class="h-4 w-4 text-white" />
  </button>
  <button
    onclick={handleMaximize}
    class="inline-flex h-[30px] w-[30px] items-center justify-center px-2 py-1 transition-colors duration-100 hover:bg-zinc-600/50"
  >
    {#if isMaximized === true}
      <Minimize class="h-4 w-4 text-white" />
    {:else}
      <Maximize class="h-4 w-4 text-white" />
    {/if}
  </button>
  <button
    onclick={() => appWindow.close()}
    class="inline-flex h-[30px] w-[30px] items-center justify-center px-2 py-1 transition-colors duration-100 hover:bg-red-700/65"
  >
    <X class="h-4 w-4 text-white" />
  </button>
</header>
<div class="flex h-screen w-screen items-center justify-center">
  {#if showCalculator}
    <main
      class="calculator flex flex-col gap-8 rounded-2xl bg-[rgba(255,255,255,0.7)] p-4 shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm"
      in:fly={{
        y: "150%",
        easing: circOut,
        duration: 400,
        delay: 100,
      }}
    >
      <div
        class="display rounded-lg bg-[rgba(255,255,255,0.9)] p-4 text-right text-[2.5rem] shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm"
      >
        {currentDisplay}
      </div>
      <div class="grid grid-cols-[repeat(4,1fr)] gap-4">
        <div class="col-[span_3] grid grid-cols-[repeat(3,1fr)] gap-4">
          <button
            onclick={clear}
            class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgba(104,104,104,0.7)] p-4 text-[2rem] text-white shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
            >{firstNumber === null && secondNumber === null
              ? "C"
              : "AC"}</button
          >
          <button
            onclick={toggleNumberNegative}
            class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgba(104,104,104,0.7)] p-4 text-[2rem] text-white shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
            >+/-</button
          >
          <button
            onclick={toggleUsingDecimal}
            class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgba(104,104,104,0.7)] p-4 text-[2rem] text-white shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
            >.</button
          >
          {#each new Array(10).fill(null).map((_, i) => 9 - i) as number}
            <button
              class:span-3={number === 0}
              class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgba(255,255,255,0.69)] p-4 text-[2rem] shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
              onclick={() => appendNumber(number)}>{number}</button
            >
          {/each}
        </div>
        <div class="grid gap-4">
          {#each operations as operation, i}
            <button
              onclick={() => setCurrentOperation(operation)}
              class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgb(255,157,0,0.7)] p-4 text-[2rem] text-white shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
              class:active-button={currentOperation === operation}
              >{@html operationsDisplay[i]}</button
            >
          {/each}
          <button
            onclick={getResult}
            class="h-fit cursor-pointer rounded-lg border-[none] bg-[rgba(255,30,0,0.7)] p-4 text-[2rem] text-white shadow-[0_4px_30px_rgba(0,0,0,0.1)] backdrop-blur-sm transition-[200ms] duration-[ease-in-out] hover:translate-y-[-3px] hover:opacity-80 active:opacity-60"
            >=</button
          >
        </div>
      </div>
    </main>
  {/if}
</div>
