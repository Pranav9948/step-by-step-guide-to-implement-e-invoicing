 
 
🟧 Step 1: Set up Basic Structure
 
 
 
 
 First lets analyse 
 
 🟡 oxalis-dist\oxalis-standalone\example.bat


 
 __________________________________________________________________________________________________________________________________________________________________________________________________________
 
 check this folder :---    oxalis-dist\oxalis-standalone\example.bat


 This batch script (example.bat) is meant to send sample invoices using the Oxalis library, a tool often 
  used for e-invoicing within the PEPPOL network, which enables electronic document exchange.

   Here's a breakdown of its content:

 (1) Script Purpose:

  The script provides two commands to send an e-invoice:

 (a) The first part sends a sample invoice to Difi's (Norway's digital administration's) test server.
 (b) The second part sends a sample invoice to a locally hosted access point.

Command Explanation:
Each java -jar command runs a Java .jar file with specific arguments to send the invoice.

Parameters:

-f: Specifies the file path of the invoice in XML format (BII04_T10_PEPPOL-v2.0_invoice.xml), following the PEPPOL standard.
-r: The receiver’s identifier in the format 9908:810418052, where 9908 is the country's code, and 810418052 is the organization’s identifier.
-s: The sender’s identifier, formatted similarly to the receiver's.
-u: The URL of the access point where the invoice will be sent. In the second example, it's http://localhost:8080/oxalis/as2, indicating a local server.
-m: The messaging protocol, here as2 (Applicability Statement 2), which is common in B2B transactions.
-i: The unique identifier for the application instance.


______________________________________________________________________________________________________

  🟡  Implementing similar process in node js 


// index.js

const fs = require('fs');
const axios = require('axios');
const path = require('path');
require('dotenv').config();

// Load environment variables
const { RECEIVER_ID, SENDER_ID, ENDPOINT_URL } = process.env;

// Path to the XML invoice file
const xmlFilePath = path.join(__dirname, 'BII04_T10_PEPPOL-v2.0_invoice.xml');

// Read XML file content
fs.readFile(xmlFilePath, 'utf-8', async (err, xmlData) => {
    if (err) {
        console.error('Error reading XML file:', err);
        return;
    }

    try {
        // Send the HTTP request to the endpoint
        const response = await axios.post(ENDPOINT_URL, xmlData, {
            headers: {
                'Content-Type': 'application/xml',
                'Receiver-ID': RECEIVER_ID,
                'Sender-ID': SENDER_ID,
            },
        });

        console.log('Invoice sent successfully:', response.data);
    } catch (error) {
        console.error('Error sending invoice:', error.response?.data || error.message);
    }
});


_____________________________________________________________________________________________________________________________________________________________________________________________


🟡 Step 2: Create Invoice Document Structure





we can see a sample invoice document in the following path:
oxalis-dist\oxalis-standalone/src/test/resources/BII04_T10_PEPPOL-v2.0_invoice.xml



________________________________________________________________________________________________________________________________________
________________________________________________________________________________________________________________________________________

🟡 Step 3: Implement Participant Handling

 next file we want to look upon is :--- ParticipantId.java 

this file is located at 

---- 

oxalis-legacy\oxalis-document-sniffer\src\main\java\network\oxalis\sniffer\identifier\ParticipantId.java


Demonstrates how sender/receiver IDs are handled



The ParticipantId class here is part of a system that handles PEPPOL participant identifiers. It provides ways to create, validate, and parse participant IDs based on PEPPOL’s ISO6523 standards. 


___________________________________________________________________________________________________


const { format } = require('util');

// Regular expression to match ISO6523 participant ID format: `xxxx:yyyy`
const ISO6523_PATTERN = /^(\\d{4}):([^\\s]+)$/;

// Max length for international organization ID
const INTERNATION_ORG_ID_MAX_LENGTH = 50;

class ParticipantId {
    /**
     * Constructor for ParticipantId using single participant ID.
     * @param {string} participantId - Participant ID string (e.g., "1234:ABCDEF").
     */
    constructor(participantId) {
        this.value = this.parse(participantId);
    }

    /**
     * Constructor for ParticipantId using schemeId and organisationId.
     * @param {Object} schemeId - Scheme ID object with a `getCode()` method.
     * @param {string} organisationId - Organization ID string.
     */
    static fromSchemeIdAndOrgId(schemeId, organisationId) {
        if (!schemeId || !schemeId.getCode) {
            throw new Error("SchemeId must have a valid ISO6523 code.");
        }

        if (!organisationId || organisationId.length > INTERNATION_ORG_ID_MAX_LENGTH) {
            throw new Error(
                `Invalid organisation ID. '${organisationId}' exceeds ${INTERNATION_ORG_ID_MAX_LENGTH} characters.`
            );
        }

        // Return new instance with formatted participant ID
        return new ParticipantId(`${schemeId.getCode()}:${organisationId}`);
    }

    /**
     * Parses a string as a participant ID in the format `xxxx:yyyy`.
     * @param {string} participantId - The participant ID string to parse.
     * @returns {string} - Validated and formatted participant ID.
     * @throws {Error} If the input doesn't match the ISO6523 pattern.
     */


    parse(participantId) {
        const trimmedId = participantId.trim().replace(/\s/g, ''); // Remove whitespace
        const match = ISO6523_PATTERN.exec(trimmedId);

        if (!match) {
            throw new Error(`ICD not found in '${participantId}'.`);
        }

        const icd = match[1];
        const organisationId = match[2];

        // Assuming SchemeId.fromISO6523(icd) fetches the correct scheme
        const schemeId = SchemeId.fromISO6523(icd);
        if (!schemeId) {
            throw new Error(`ICD ${icd} is unknown.`);
        }

        return `${schemeId.getCode()}:${organisationId}`;
    }

    /**
     * Static method to create a validated ParticipantId instance.
     * @param {string} participantId - The participant ID string to validate.
     * @returns {ParticipantId} - Instance of ParticipantId.
     */
    static valueOf(participantId) {
        return new ParticipantId(participantId);
    }

    /**
     * Checks if a given string matches the participant ID pattern.
     * @param {string} value - The string to validate.
     * @returns {boolean} - True if the pattern matches, false otherwise.
     */
    static isValidParticipantIdentifierPattern(value) {
        if (!value) return false;
        return ISO6523_PATTERN.test(value);
    }

    /**
     * Converts ParticipantId to its string representation.
     * @returns {string} - The participant ID string.
     */
    toString() {
        return this.value;
    }

    /**
     * Converts ParticipantId to a ParticipantIdentifier object.
     * @returns {Object} - The `ParticipantIdentifier` representation.
     */
    toVefa() {
        return ParticipantIdentifier.of(this.value, ParticipantIdentifier.DEFAULT_SCHEME);
    }

    /**
     * Equality check to compare two ParticipantId instances.
     * @param {ParticipantId} other - Another ParticipantId instance.
     * @returns {boolean} - True if equal, false otherwise.
     */
    equals(other) {
        return other instanceof ParticipantId && this.value === other.value;
    }
}

module.exports = ParticipantId;



    /**
     * Parses a string as a participant ID in the format `xxxx:yyyy`.
     * @param {string} participantId - The participant ID string to parse.
     * @returns {string} - Validated and formatted participant ID.
     * @throws {Error} If the input doesn't match the ISO6523 pattern.
     */


 _____________________________________________________________________________________________________________________________________
_________________________________________________________________________________________________________________________________________

🟡  Step 4: Document Wrapping

Inorder to understand the document structure 

check this file :-  For document structure understanding:

oxalis-legacy\oxalis-document-sniffer\src\main\java\network\oxalis\sniffer\sbdh\SbdhWrapper.java


SbdhWrapper.java - Shows how documents are wrapped with required headers


The SbdhWrapper Java class takes a document (represented as an input stream) and headers, then wraps them into a StandardBusinessDocument format, which is commonly used in business document exchanges like e-invoicing


________________________________________________________________________________________________________________________________

Node.js Implementation of SbdhWrapper

In Node.js, we'll leverage libraries like xml2js for XML handling and stream for processing the input stream. 


// SbdhWrapper.js

const { Readable, pipeline } = require('stream');
const { parseStringPromise, Builder } = require('xml2js');
const { promisify } = require('util');

const pipelineAsync = promisify(pipeline);

class SbdhWrapper {
    constructor() {}

    /**
     * Wraps the payload and headers into a StandardBusinessDocument format.
     *
     * @param {Readable} inputStream - The document input stream to be wrapped.
     * @param {Object} headers - Headers to include in the SBDH.
     * @returns {Promise<Buffer>} - The wrapped document in UTF-8 encoded byte buffer.
     */
    async wrap(inputStream, headers) {
        const outputStream = new Readable();

        // Build the SBDH XML with headers
        const sbdh = this._buildSbdhXml(headers);

        // Transform input stream to a buffer
        const payloadBuffer = await this._streamToBuffer(inputStream);

        // Create the SBD document combining SBDH with the payload
        const sbdDocument = this._combineSbdhWithPayload(sbdh, payloadBuffer.toString('utf-8'));

        // Push the result to outputStream
        outputStream.push(sbdDocument);
        outputStream.push(null); // Signals the end of the stream

        return Buffer.from(sbdDocument, 'utf-8');
    }

    // Creates the SBDH XML structure from headers
    _buildSbdhXml(headers) {
        const sbdhStructure = {
            StandardBusinessDocument: {
                StandardBusinessDocumentHeader: {
                    ...headers,
                },
                Payload: {
                    Data: '', // This will be the document content
                },
            },
        };

        const builder = new Builder();
        return builder.buildObject(sbdhStructure);
    }

    // Combines the SBDH XML and payload XML into a single document
    _combineSbdhWithPayload(sbdhXml, payloadXml) {
        return sbdhXml.replace('<Data></Data>', `<Data>${payloadXml}</Data>`);
    }

    // Helper to convert stream to buffer
    async _streamToBuffer(stream) {
        const chunks = [];
        for await (const chunk of stream) {
            chunks.push(chunk);
        }
        return Buffer.concat(chunks);
    }
}

module.exports = SbdhWrapper;




_____________________________________________________________________________________________________________________________________________________________________________________________

🟡 Step 6: Implement Transmission

Next examine the transmission flow:

check this folder :- oxalis-api\src\main\java\network\oxalis\api\outbound\Transmitter.java

Shows the core interface for sending documents

 This transmitter is responsible for sending  e-invoice messages using different protocols depending on the transport profile.


 ____________________________________________________________________________________________

 conversion of that code into node js

 __________________________________________________________________________________



 // transmitter.js
const axios = require('axios');
const { initTracer } = require('opentracing');

// Custom error for transmission failures
class TransmissionException extends Error {
    constructor(message) {
        super(message);
        this.name = 'TransmissionException';
    }
}

class Transmitter {
    constructor() {
        // Initialize a basic tracer (could be connected to a real tracing system)
        this.tracer = initTracer();
    }

    // Transmit method without tracing
    async transmit(transmissionMessage) {
        try {
            const response = await axios.post(transmissionMessage.url, transmissionMessage.data, {
                headers: transmissionMessage.headers,
            });
            return response.data; // TransmissionResponse equivalent
        } catch (error) {
            throw new TransmissionException('Transmission failed: ' + error.message);
        }
    }

    // Transmit method with tracing
    async transmitWithTracing(transmissionMessage, span) {
        const traceSpan = span || this.tracer.startSpan('transmit');
        try {
            const response = await this.transmit(transmissionMessage);
            traceSpan.setTag('status', 'success');
            return response;
        } catch (error) {
            traceSpan.setTag('error', true);
            throw new TransmissionException('Transmission failed: ' + error.message);
        } finally {
            traceSpan.finish();
        }
    }
}

module.exports = Transmitter;


__________________________________________

explaination

_________________________________________


This code defines a Transmitter class in JavaScript that is responsible for sending HTTP requests with optional tracing capabilities. 

initTracer: This function from the opentracing library initializes a tracer, which is helpful for tracking and logging operations in distributed systems.


Usage


This Transmitter class could be used to send data to another server or endpoint, while also tracking the request's lifecycle if tracing is enabled. The tracing helps in monitoring, especially when multiple services or requests are involved.

_________________________________

Explanation

_________________________________


Transmitter Class

The main class, Transmitter, has methods for sending messages and a way to handle tracing (to monitor the path of the request).

Constructor:
Initializes the tracer, which can track requests for debugging and performance monitoring.

transmit Method:

Input: A transmissionMessage object containing a url, data, and optional headers.


transmitWithTracing Method:

Purpose: Adds tracing to the transmit method.
Steps:
(i) Starts a tracing span (a unit of work in tracing).
(ii) Calls transmit() and sets a status tag to "success" if it works.
(iii) If it fails, sets the error tag on the trace.
(iv) Finally: Ends the trace span.


_________________________________________________________________________________________________
_________________________________________________________________________________________________

 Lets now explore how this interface is implemented and the TransmissionMessage structure next


From the codebase, we can see there are two main implementations of the Transmitter interface mentioned in the release notes:

(i) SimpleTransmitter - Basic implementation that just sends the message
(ii) EvidencePerisistingTransmitter - More advanced implementation that also stores transmission evidence in a database using messageId as lookup key




The transmission flow works like this:

The TransmissionMessage contains:

    (1) The actual payload (invoice document)
    (2) Headers with sender/receiver information
    (3) Document type identifiers
    (4) Profile information

The transmission process involves:

 (1) Wrapping the document with SBDH (Standard Business Document Header) using SbdhWrapper
 (2) Validiting all required fields through TransmissionRequestBuilder
 (3) Looking up endpoints through SMP if not explicitly provided
 (4) Sending the message using the configured protocol (AS2)
 (5) Returning a TransmissionResponse with result5


This can be found here in 

 oxalis-outbound\src\main\java\network\oxalis\outbound\transmission\TransmissionRequestBuilder.java


This handles validation and construction of the transmission request.

This Java class, TransmissionRequestBuilder, is part of an e-invoicing framework. It’s designed to build and manage a transmission request in a standardized format for the PEPPOL e-invoicing network. 


🟢  Step 5: Build Transmission Request

TransmissionRequestBuilder handles the preparation of a TransmissionRequest by setting up various configurations such as payload, sender/receiver, document types, and header information. It also handles custom header overrides and tracing.


TransmissionRequestBuilder configures and validates all the data needed to send a standardized PEPPOL transmission request. It’s flexible, allowing for manual overrides, custom header merging, and error handling, making it essential for managing the structured requirements of e-invoicing over PEPPOL.




-------------------------------------------------------------
-------------------------------------------------------------

Node.js Implementation of TransmissionRequestBuilder


// TransmissionRequestBuilder.js

const axios = require('axios');
const { initTracer } = require('opentracing');

class TransmissionRequestBuilder {
    constructor({ contentDetector, lookupService, tagGenerator, headerParser, tracer }) {
        this.contentDetector = contentDetector;       // ContentDetector: Detects content type in the payload
        this.lookupService = lookupService;           // Service to look up endpoint based on identifiers
        this.tagGenerator = tagGenerator;             //  Generates metadata tags to describe the request.
        this.headerParser = headerParser;             // Parses headers from the payload
        this.tracer = tracer || initTracer();         // Initializes tracer (optional for tracing)
        
        this.allowOverride = false;                   // Flag to allow header overrides
        this.payload = null;                          // Holds the document to be transmitted
        this.endpoint = null;                         // Endpoint URL for transmission
        this.suppliedHeaderFields = {};               // Header fields supplied by the caller
        this.effectiveHeaderFields = null;            // Merged effective header fields
        this.tag = 'NONE';                            // Optional tag
    }

    // Sets payload

    setPayload(inputStream) {
        this.payload = inputStream;
        return this;
    }

    // Sets endpoint manually

    overrideEndpoint(endpoint) {
        this.endpoint = endpoint;
        return this;
    }

    // Setters for transmission metadata

    receiver(receiverId) {
        this.suppliedHeaderFields.recipientId = receiverId;
        return this;
    }

    sender(senderId) {
        this.suppliedHeaderFields.senderId = senderId;
        return this;
    }

    documentType(documentTypeId) {
        this.suppliedHeaderFields.documentTypeIdentifier = documentTypeId;
        return this;
    }

    processType(processTypeId) {
        this.suppliedHeaderFields.profileTypeIdentifier = processTypeId;
        return this;
    }

    tag(tag) {
        this.tag = tag;
        return this;
    }

    // Builds the Transmission Request with optional tracing
    async build(rootSpan) {
        const span = this.tracer.startSpan('build', { childOf: rootSpan });
        try {
            // Verify payload and headers
            if (!this.payload) throw new Error('Payload is missing');

            // Merge headers from payload with supplied headers
            this.effectiveHeaderFields = this._mergeHeaders();

            // Look up endpoint if not manually set
            if (!this.endpoint || !this.allowOverride) {
                this.endpoint = await this.lookupService.lookup(this.effectiveHeaderFields);
            }

            // Tag metadata (example structure)
            const metadata = this.tagGenerator.generate('OUT', this.tag);

            // Create and return the transmission request object
            return {
                endpoint: this.endpoint,
                payload: this.payload,
                headers: this.effectiveHeaderFields,
                metadata,
            };
        } catch (error) {
            span.setTag('error', true);
            throw new Error('Failed to build TransmissionRequest: ' + error.message);
        } finally {
            span.finish();
        }
    }

    // Merges provided headers with those from the payload, allowing overrides if enabled
    _mergeHeaders() {
        const parsedHeaders = this.headerParser.parse(this.payload);
        
        // Merge parsed headers with supplied ones, favoring supplied values
        const mergedHeaders = { ...parsedHeaders, ...this.suppliedHeaderFields };
        
        if (!this.allowOverride) {
            this._checkRestrictedHeaders(parsedHeaders, mergedHeaders);
        }
        
        return mergedHeaders;
    }

    // Checks if restricted headers are overridden (based on production rules)
    _checkRestrictedHeaders(parsedHeaders, mergedHeaders) {
        const restrictedHeaders = ['senderId', 'recipientId', 'documentTypeIdentifier', 'profileTypeIdentifier'];
        const overriddenHeaders = restrictedHeaders.filter(
            (header) => parsedHeaders[header] && mergedHeaders[header] && parsedHeaders[header] !== mergedHeaders[header]
        );

        if (overriddenHeaders.length > 0) {
            throw new Error(`Restricted headers overridden: ${overriddenHeaders.join(', ')}`);
        }
    }

    // Wrapper function to serialize the payload with necessary headers (similar to SBDH wrapping)
    wrapPayloadWithHeaders() {
        // Wrap payload data (this would need to be implemented as per protocol specifications)
        return Buffer.concat([Buffer.from(JSON.stringify(this.effectiveHeaderFields)), this.payload]);
    }
}

module.exports = TransmissionRequestBuilder;


____________________________________________________________________________________________________________
_____________________________________________________________________________________________________________

To use this TransmissionRequestBuilder

// exampleUsage.js

const TransmissionRequestBuilder = require('./TransmissionRequestBuilder');

// Initialize required services (pseudo-code)
const contentDetector = new ContentDetector();
const lookupService = new LookupService();
const tagGenerator = new TagGenerator();
const headerParser = new HeaderParser();
const tracer = initTracer();

// Instantiate the builder
const builder = new TransmissionRequestBuilder({
    contentDetector,
    lookupService,
    tagGenerator,
    headerParser,
    tracer,
});

// Set up transmission request
builder.setPayload(Buffer.from("Sample payload data"))
    .sender('Sender123')
    .receiver('Receiver456')
    .documentType('Invoice')
    .processType('Standard')
    .tag('InvoiceSubmission');

// Build the request
(async () => {
    try {
        const transmissionRequest = await builder.build();
        console.log('Transmission Request:', transmissionRequest);
    } catch (error) {
        console.error('Error building transmission request:', error.message);
    }
})();



This setup creates a customizable transmission request with flexible endpoint management, header merging, and optional tracing, making it suitable for e-invoicing 

