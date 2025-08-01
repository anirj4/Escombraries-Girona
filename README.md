// Alexa Skills Kit SDK for Node.js
const Alexa = require('ask-sdk-core');

// Dades dels horaris de recollida de residus
const horariResidus = {
  "dilluns": ["orgànica", "resta", "tèxtil sanitari"],
  "dimarts": ["orgànica"],
  "dimecres": ["envasos", "tèxtil sanitari"],
  "dijous": ["orgànica", "paper i cartró"],
  "divendres": ["orgànica", "resta", "tèxtil sanitari"],
  "dissabte": ["envasos"],
  "diumenge": ["orgànica"]
};

// Noms dels dies de la setmana en català
const diesSetmana = ["diumenge", "dilluns", "dimarts", "dimecres", "dijous", "divendres", "dissabte"];

// --- Helper Functions ---

/**
 * Funció per obtenir el nom del dia de la setmana.
 * @param {number} offset - 0 per avui, 1 per demà, etc.
 * @returns {string} El nom del dia de la setmana en minúscules.
 */
function getDayOfWeek(offset = 0) {
  const today = new Date();
  // Ajust per la zona horària de Madrid (CET/CEST)
  const targetDate = new Date(today.toLocaleString("en-US", { timeZone: "Europe/Madrid" }));
  targetDate.setDate(targetDate.getDate() + offset);
  return diesSetmana[targetDate.getDay()];
}

/**
 * Formata un array de residus en una cadena de text llegible.
 * Ex: ["a", "b", "c"] -> "a, b i c"
 * @param {string[]} array - Llista de residus.
 * @returns {string} La cadena de text formatada.
 */
function formatList(array) {
  if (!array || array.length === 0) {
    return "res en particular";
  }
  if (array.length === 1) {
    return array[0];
  }
  if (array.length === 2) {
    return `${array[0]} i ${array[1]}`;
  }
  const last = array.pop();
  return `${array.join(', ')} i ${last}`;
}


// --- Intent Handlers ---

// Handler per quan l'usuari llança la skill sense un intent específic.
const LaunchRequestHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
  },
  handle(handlerInput) {
    const speakOutput = 'Benvingut a Residus Girona. Pots preguntar-me què pots llençar avui, demà o quan es llença un tipus de residu concret.';
    return handlerInput.responseBuilder
      .speak(speakOutput)
      .reprompt(speakOutput)
      .getResponse();
  }
};

// Handler per a la intenció de saber què llençar avui.
const QueTirarHoyIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest' &&
      Alexa.getIntentName(handlerInput.requestEnvelope) === 'QueTirarHoyIntent';
  },
  handle(handlerInput) {
    const dia = getDayOfWeek(0);
    const residusDelDia = horariResidus[dia];
    const llistaResidus = formatList(residusDelDia);
    const speakOutput = `Avui ${dia}, pots llençar ${llistaResidus}.`;

    return handlerInput.responseBuilder
      .speak(speakOutput)
      .getResponse();
  }
};

// Handler per a la intenció de saber què llençar demà.
const QueTirarMananaIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest' &&
      Alexa.getIntentName(handlerInput.requestEnvelope) === 'QueTirarMananaIntent';
  },
  handle(handlerInput) {
    const dia = getDayOfWeek(1);
    const residusDelDia = horariResidus[dia];
    const llistaResidus = formatList(residusDelDia);
    const speakOutput = `Demà ${dia}, podràs llençar ${llistaResidus}.`;

    return handlerInput.responseBuilder
      .speak(speakOutput)
      .getResponse();
  }
};

// Handler per a la intenció de saber quan llençar un tipus de residu.
const CuandoTirarResiduoIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest' &&
      Alexa.getIntentName(handlerInput.requestEnvelope) === 'CuandoTirarResiduoIntent';
  },
  handle(handlerInput) {
    const residuSlot = Alexa.getSlot(handlerInput.requestEnvelope, 'residu');
    let speakOutput = '';

    if (residuSlot && residuSlot.value) {
      const residu = residuSlot.value.toLowerCase();

      // Cas especial per al vidre
      if (residu === 'vidre') {
        speakOutput = "El vidre no té un dia de recollida porta a porta. El pots dipositar al contenidor verd qualsevol dia de la setmana.";
      } else {
        const diesDeRecollida = [];
        for (const dia in horariResidus) {
          // Ajust per incloure "paper i cartró" quan es busca per "paper" o "cartró"
          if (horariResidus[dia].some(r => r.includes(residu))) {
            diesDeRecollida.push(dia);
          }
        }

        if (diesDeRecollida.length > 0) {
          const llistaDies = formatList(diesDeRecollida);
          speakOutput = `Pots llençar ${residu} el${diesDeRecollida.length > 1 ? 's' : ''} ${llistaDies}.`;
        } else {
          speakOutput = `Ho sento, no he trobat informació sobre la recollida de ${residu}. Prova amb un altre tipus.`;
        }
      }
    } else {
      speakOutput = 'Quin tipus de residu vols consultar?';
    }

    return handlerInput.responseBuilder
      .speak(speakOutput)
      .reprompt('Pots preguntar-me per un altre residu si vols.')
      .getResponse();
  }
};

// Handlers estàndard (Ajuda, Cancel·lar, Aturar, Final de sessió)
const HelpIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest' &&
      Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.HelpIntent';
  },
  handle(handlerInput) {
    const speakOutput = 'Pots dir-me: "què llenço avui", "què toca demà" o "quan puc llençar el paper". Com et puc ajudar?';
    return handlerInput.responseBuilder
      .speak(speakOutput)
      .reprompt(speakOutput)
      .getResponse();
  }
};

const CancelAndStopIntentHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest' &&
      (Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.CancelIntent' ||
        Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.StopIntent');
  },
  handle(handlerInput) {
    const speakOutput = 'Adéu!';
    return handlerInput.responseBuilder
      .speak(speakOutput)
      .getResponse();
  }
};

const SessionEndedRequestHandler = {
  canHandle(handlerInput) {
    return Alexa.getRequestType(handlerInput.requestEnvelope) === 'SessionEndedRequest';
  },
  handle(handlerInput) {
    console.log(`Sessió finalitzada amb motiu: ${handlerInput.requestEnvelope.request.reason}`);
    return handlerInput.responseBuilder.getResponse();
  }
};

// Handler per a intents no reconeguts o errors. Ha d'anar al final.
const ErrorHandler = {
  canHandle() {
    return true;
  },
  handle(handlerInput, error) {
    console.log(`Error gestionat: ${error.stack}`);
    const speakOutput = 'Ho sento, he tingut un problema. Si us plau, intenta-ho de nou.';

    return handlerInput.responseBuilder
      .speak(speakOutput)
      .reprompt(speakOutput)
      .getResponse();
  }
};

// Constructor de la skill
exports.handler = Alexa.SkillBuilders.custom()
  .addRequestHandlers(
    LaunchRequestHandler,
    QueTirarHoyIntentHandler,
    QueTirarMananaIntentHandler,
    CuandoTirarResiduoIntentHandler,
    HelpIntentHandler,
    CancelAndStopIntentHandler,
    SessionEndedRequestHandler
  )
  .addErrorHandlers(
    ErrorHandler
  )
  .lambda();
