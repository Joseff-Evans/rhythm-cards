<script lang="ts">
  import "./main.css"

  const degree = ["I", "II", "III", "IV", "V", "VI", "VII"];
  const note = [
    "C", "C♯/D♭", "D", "D♯/E♭", "E", "F",
    "F♯/G♭", "G", "G♯/A♭", "A", "A♯/B♭", "B"
  ];
  const beat = [
    "0000", "0001", "0010", "0011",
    "0100", "0101", "0110", "0111",
    "1000", "1001", "1010", "1011",
    "1100", "1101", "1110", "1111"
  ];
  const type = ["M", "m"];

  const random = (arr: string[]): string =>
    arr[Math.floor(Math.random() * arr.length)];

  const randomCard = () => ({
    degree: random(degree),
    note: random(note),
    beat: random(beat),
    type: random(type)
  });

  const generateCards = (count: number) =>
    Array.from({ length: count }, randomCard);

  let cards = $state<ReturnType<typeof randomCard>[]>([]);
  let number = $state(2);

  function regenerate() {
    cards = generateCards(Math.max(0, Math.min(500, Math.floor(Number(number) || 0))));
  }
  let draggedIndex = $state<number | null>(null);
let dragOverIndex = $state<number | null>(null);

function handleDragStart(event: DragEvent, index: number) {
  draggedIndex = index;

  if (event.dataTransfer) {
    event.dataTransfer.effectAllowed = "move";
    event.dataTransfer.setData("text/plain", String(index));
  }
}

function handleDragOver(event: DragEvent, index: number) {
  event.preventDefault();

  if (event.dataTransfer) {
    event.dataTransfer.dropEffect = "move";
  }

  dragOverIndex = index;
}

function handleDrop(event: DragEvent, targetIndex: number) {
  event.preventDefault();

  if (draggedIndex === null || draggedIndex === targetIndex) {
    draggedIndex = null;
    dragOverIndex = null;
    return;
  }

  const reordered = [...cards];
  const [movedCard] = reordered.splice(draggedIndex, 1);
  reordered.splice(targetIndex, 0, movedCard);

  cards = reordered;
  draggedIndex = null;
  dragOverIndex = null;
}

function handleDragEnd() {
  draggedIndex = null;
  dragOverIndex = null;
}

function getWavePaths(card: {
  note: string;
  degree: string;
  type: string;
  beat: string;
}) {
  let seed = 0;
  const key = `${card.note}-${card.degree}-${card.type}-${card.beat}`;

  for (let i = 0; i < key.length; i++) {
    seed = (Math.imul(seed, 31) + key.charCodeAt(i)) >>> 0;
  }

  const random = () => {
    seed = (Math.imul(seed, 1664525) + 1013904223) >>> 0;
    return seed / 4294967296;
  };

  const activeBeats = [...card.beat].map(Number);
  const paths: string[] = [];
  const layers = 5 + Math.floor(random() * 4);
  const frequency = 1.2 + random() * 2.5;
  const phase = random() * Math.PI * 2;
  const amplitude = 8 + random() * 12;

  for (let layer = 0; layer < layers; layer++) {
    const baseY = 25 + layer * (50 / layers);
    const layerPhase = phase + layer * 0.24;
    const layerAmplitude = amplitude * (0.5 + random() * 0.5);
    const points: string[] = [];

    for (let x = 0; x <= 200; x += 2) {
      const position = x / 200;
      const beatIndex = Math.min(
        3,
        Math.floor(position * 4)
      );

      // Active beats create stronger pulses.
      const pulse = activeBeats[beatIndex] ? 1 : 0.3;

      const wave =
        Math.sin(position * Math.PI * 2 * frequency + layerPhase) *
        layerAmplitude *
        pulse;

      const detail =
        Math.sin(position * Math.PI * 8 + layerPhase) *
        layerAmplitude *
        0.12;

      const y = baseY + wave + detail;

      points.push(
        `${x === 0 ? "M" : "L"}${x},${y.toFixed(2)}`
      );
    }

    paths.push(points.join(" "));
  }

  return paths;
}
</script>

<main class="mainContainer">
  <header class="pageHeader">
    <span class="eyebrow">AN EXERCISE IN CHANCE</span>
    <h1>Chance Cards<span>.</span></h1>
    <p class="subtitle">A deck of prompts for making something unexpected.</p>
  </header>

  <section class="controls">
    <label for="cardCount">Number of cards</label>
    <input id="cardCount" type="number" min="1" max="500" bind:value={number} />
    <button onclick={regenerate}>
      <span>↻</span> Generate cards
    </button>
  </section>

  {#if cards.length > 0}
    <div class="resultsHeader">
      <span>Your deck</span>
      <span class="count">{cards.length} cards</span>
    </div>

<div class="cardContainer">
  {#each cards as card, i (card)}
    <article
      class="card"
      class:dragging={draggedIndex === i}
      class:dragOver={dragOverIndex === i && draggedIndex !== i}
      draggable="true"
      ondragstart={(event) => handleDragStart(event, i)}
      ondragover={(event) => handleDragOver(event, i)}
      ondrop={(event) => handleDrop(event, i)}
      ondragend={handleDragEnd}
    ><svg
  class="wavePattern"
  viewBox="0 0 200 100"
  preserveAspectRatio="none"
  aria-hidden="true"
>
  {#each getWavePaths(card) as path, j}
    <path
      d={path}
      class:waveAccent={j % 3 === 0}
      style={`--layer: ${j}`}
    />
  {/each}
</svg>
      <div class="cardHeader">
        <span class="cardIndex">{String(i + 1).padStart(2, "0")}</span>
        <span class="degree">{card.degree}</span>
      </div>

      <div class="cardNote">
        {card.note}
        <span class:minor={card.type === "m"} class="chordType">
          {card.type === "M" ? "maj" : "min"}
        </span>
      </div>

      <div class="cardFooter">
        <div class="rhythm">
          {#each card.beat.split("") as bit, j}
            <span class:active={bit === "1"} class="beatDot"></span>
          {/each}
        </div>
        <span class="rhythmText">{card.beat}</span>
      </div>
    </article>
  {/each}
</div>

  {:else}
    <div class="emptyState">
      <h2>Your deck starts here.</h2>
      <p>Choose how many cards you want, then generate your first set.</p>
    </div>
  {/if}
</main>