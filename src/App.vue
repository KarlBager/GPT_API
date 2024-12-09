<script setup>
import { ref } from 'vue';
import { createCompletionsChat } from './text.js';
import { createInitialCompletionsChat } from './text.js';
import { createImagePrompt } from './text.js';
import { getRandomInt } from './text.js';
import names from './names.json';
import { startBackground } from './background.js';

import { textToImage, saveBase64AsImageFile } from './stable.js'

startBackground();



const content = ref('');
const chatHistory = ref([]);
const persona = ref(''); // To store the selected persona
const systemPrompt = "You have chosen a person that you must take on as your persona. Your identity is a secret that the user must guess by asking you questions. The user will communicate in English or Danish so please respond in one of these languages. Your goal is to answer these questions truthfully but without revealing too much too soon – and NEVER the name itself. Refuse to answer questions that aren't yes/no questions. Respond in a way that offers helpful hints and insight, but only provide specific details once the user is reasonably close to guessing who you are.\n\n" +
"Rules of the Game:\n" +
"1. Respond to each question accurately, but don’t volunteer extra information.\n" +
"2. If the user asks about general attributes (such as gender, field of fame, or era), respond directly but vaguely unless it’s obvious.\n" +
"3. As the user’s questions become more targeted (like asking about specific achievements, well-known events, or places), you can provide slightly more precise hints to encourage them.\n" +
"4. Once the user is very close (e.g., guessing correctly or almost correctly), confirm with a more explicit clue or reveal.\n\n" +
"Playfully encourage them as they get closer, and subtly guide them toward the answer without giving away who you are too easily!";


const imagePrompt = ref('')
const imgSrc = ref('')



// Function to choose a persona when the app loads
async function choosePersona() {

  const initialPrompt = "Please think of a completely random famous person that the average person might recognize. After selecting a name, respond with only the name and no additional information.";
  try {
    //const response = await createInitialCompletionsChat(initialPrompt); // Get the response
    //persona.value = response.value; // Store the selected persona

    let arrayLength = names.length;
    let nameIndex = getRandomInt(arrayLength);
    console.log(nameIndex, names[nameIndex]);
    persona.value = names[nameIndex];
    console.log(persona.value);

    chatHistory.value.push({ role: 'assistant', content: `I am thinking of someone famous. You can start asking questions!` });
    //console.log(persona.value);

    await createCompletionsChat(systemPrompt); // Send the system prompt
  } catch (error) {
    console.error("Error selecting persona:", error);
    chatHistory.value.push({ role: 'assistant', content: 'Sorry, I could not choose a persona.' });
  }
}



// Function to handle user input
async function handleUserInput() {
  const userInput = content.value.trim();
  
  if (!userInput) {
    alert("Please enter a question or a guess.");
    return;
  }

  // Add user message to chat history
  chatHistory.value.push({ role: 'user', content: userInput });

  // Construct the full prompt including the persona and chat history
  const promptMessages = chatHistory.value
    .map(msg => `${msg.role === 'user' ? 'User' : 'Assistant'}: ${msg.content}`)
    .join('\n');

  const fullPrompt = `${persona.value}:\n${promptMessages}\nAssistant:`;

  try {
    const response = await createCompletionsChat(fullPrompt); // Send the prompt to the model
    chatHistory.value.push({ role: 'assistant', content: response.value }); // Add assistant response to chat history
    content.value = ''; // Clear input field
  } catch (error) {
    console.error("Error fetching AI response:", error);
    chatHistory.value.push({ role: 'assistant', content: 'Sorry, an error occurred!' });
  }
}



async function generateImage() {
  if (!imagePrompt.value) return
  const image = await textToImage(imagePrompt.value);
  imgSrc.value = image;
  saveBase64AsImageFile(image, 'generatedImage.png');
}



// Function to handle user input
async function requestImageHint() {
  const imageSystemPrompt = "The user has requested an image hint. With the past conversation in mind, you will answer this with a prompt for an image generator AI that contains a few elements that can hint at the chosen person. You are allowed to include a person, but put more emphasis on relevant props or settings. Answer only the prompt itself";


// Construct the full prompt including the persona and chat history
const promptMessages = chatHistory.value
  .map(msg => `${msg.role === 'user' ? 'User' : 'Assistant'}: ${msg.content}`)
  .join('\n');

const fullPrompt = `${persona.value}:\n${promptMessages}\nSystem: ${imageSystemPrompt}\nAssistant:`;


  try {
    const response = await createImagePrompt(fullPrompt); // Send the prompt to the model
    console.log(response.value);

    imagePrompt.value = response.value;
    generateImage();
    

  } catch (error) {
    console.error("Error fetching AI response:", error);
    chatHistory.value.push({ role: 'assistant', content: 'Sorry, an error occurred!' });
  }
}




// On component mount, choose a persona
choosePersona();
</script>

<template>
     <div class="container">
    <h1>Guess the <strong>Person</strong></h1>

    <div class="chat-window">
      <div v-for="(message, index) in chatHistory" :key="index" :class="message.role">
        <p>{{ message.content }}</p>
      </div>
      <div>
        <img class="hintImage" :src="imgSrc" v-if="imgSrc" />
      </div>
    </div>

    

    <input
      type="text"
      class="input"
      placeholder="Ask me a question or take a guess..."
      v-model="content"
      autofocus
      @keyup.enter="handleUserInput"
    />

    <button @click="requestImageHint">Image Hint</button>
  </div>

</template>

<style>
body{
background-color: slategray;
}

h1{
  text-align: center;
  font-weight: 200;
  font-size: 2rem;
}

h1 strong{
  font-weight: 800;
}

.input{
background-color: rgba(0,0,0,0);
color: white;
border: none;
width: 30vw;
margin: 0 auto;
height: 2rem;
}

.input::placeholder{
color: #d0cece;
}

.container{
  margin: 0 auto;
  text-align: center;
  padding-top: 3rem;
}


.chat-window {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 100vh;
  overflow-y: auto;
  margin-bottom: 20px;
}

.user {
  font-weight: 300;
}

.assistant {
  font-weight: 800;
}


button {
  height: 2rem;
  background: none;
  border: 1px solid #f4f4f4;
  color: #f4f4f4;
}

p, h1{
color: #f4f4f4;
}

.hintImage{
  max-width: 30rem;

}


</style>
