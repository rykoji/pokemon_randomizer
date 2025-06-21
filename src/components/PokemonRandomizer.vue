<script setup>
import { ref, onMounted } from 'vue'

const pokemons = ref([])
const error = ref(null)
const loading = ref(true)

// Game filtering
const selectedGame = ref('') // blank means all games / national
const games = ref([{ label: 'All', value: '' }])

const formatLabel = (name) => name.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase())

const loadGames = async () => {
  try {
    const res = await fetch('https://pokeapi.co/api/v2/pokedex')
    const data = await res.json()
    const gameOptions = data.results.map(p => ({ label: formatLabel(p.name), value: p.name }))
    games.value.push(...gameOptions)
  } catch (err) {
    console.error('Failed to load games', err)
  }
}

const POKEBALL_IMAGE = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/items/poke-ball.png'

const getRandomNumber = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

const setPlaceholderPokeballs = () => {
  pokemons.value = Array.from({ length: 6 }, (_, index) => ({
    id: index + 1,
    name: 'Pokémon',
    sprites: { front_default: POKEBALL_IMAGE },
    types: [{ type: { name: '???' } }]
  }))
  loading.value = false
}

const fetchRandomPokemons = async () => {
  try {
    loading.value = true

    let responses = []

    if (!selectedGame.value) {
      // No game selected: randomize across all available Pokémon IDs (1..898)
      const pokemonIds = Array.from({ length: 6 }, () => getRandomNumber(1, 898))
      responses = await Promise.all(
        pokemonIds.map(id => fetch(`https://pokeapi.co/api/v2/pokemon/${id}`).then(r => r.json()))
      )
    } else {
      // 1. Retrieve the pokedex for the selected game
      const pokedexResponse = await fetch(`https://pokeapi.co/api/v2/pokedex/${selectedGame.value}`)
      const pokedexData = await pokedexResponse.json()
      const entries = pokedexData.pokemon_entries

      if (!entries || entries.length === 0) {
        throw new Error('No Pokémon found for selected game')
      }

      // 2. Choose up to 6 unique random entries
      const randomIndexes = new Set()
      while (randomIndexes.size < 6 && randomIndexes.size < entries.length) {
        randomIndexes.add(getRandomNumber(0, entries.length - 1))
      }

      // 3. Fetch Pokémon data for the chosen species
      responses = await Promise.all(
        Array.from(randomIndexes).map(idx => {
          const entry = entries[idx]
          const name = entry.pokemon_species.name
          const entryNumber = entry.entry_number
          return fetch(`https://pokeapi.co/api/v2/pokemon/${name}`)
            .then(r => r.json())
            .then(data => ({ ...data, entry_number: entryNumber }))
        })
      )
    }

    pokemons.value = responses
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}

// Fetch initial random Pokémon
onMounted(async () => {
  setPlaceholderPokeballs()
  await loadGames()
})
</script>

<template>
  <div class="flex flex-col items-center justify-center h-screen max-w-3xl mx-auto p-5">
    <h2 class="mb-4 flex items-center justify-center">Randomizer Pokémon</h2>
    <div v-if="error">
      <p>Error: {{ error }}</p>
    </div>
    <div v-else class="pokemon-grid">
      <div v-for="pokemon in pokemons" :key="pokemon.id" class="pokemon-card">
        <img :src="pokemon.sprites.front_default" :alt="pokemon.name" />
        <h3><strong>#{{ pokemon.entry_number || pokemon.id }} - {{ pokemon.name }}</strong></h3>
        <p>Type: {{ pokemon.types.map(type => type.type.name).join(', ') }}</p>
      </div>
    </div>
    <div class="controls">
      <select v-model="selectedGame" class="game-select">
        <option v-for="game in games" :key="game.value" :value="game.value">{{ game.label }}</option>
      </select>
      <button 
        class="pokemon-button"
        @click="fetchRandomPokemons"
      >
        Randomize Pokémon
      </button>
    </div>
  </div>
</template>

<style scoped>
.pokemon-list {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  margin: 0 auto;
  padding: 20px;
}

.controls {
  width: 100%;
  text-align: center;
  margin-top: 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.pokemon-button {
  background-color: #4CAF50;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s ease;
}

.pokemon-button:hover {
  background-color: #45a049;
}

.pokemon-button:active {
  transform: scale(0.98);
}

.game-select {
  display: flex ;
  padding: 8px 12px;
  margin-bottom: 1rem;
  border-radius: 4px;
  border: 1px solid #ccc;
}

.pokemon-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
  width: 100%;
}

@media (max-width: 640px) {
  .pokemon-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

.pokemon-card {
  background: #f5f5f5;
  border-radius: 8px;
  padding: 15px;
  text-align: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.pokemon-card img {
  width: 100px;
  height: 100px;
  margin-bottom: 10px;
}

.pokemon-card h3 {
  margin: 0;
  font-size: 1.2em;
  color: #333;
}

.pokemon-card p {
  margin: 5px 0 0;
  color: #666;
}
</style>
