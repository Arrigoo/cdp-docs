# Tracking script Arrigoo CDP

The frontend tracking script for the Arrigoo CDP essentially adds an object to the window object called window.argo and adds some events to the global scope of the page that are dispatched by the script.

## Events

The events introduced are:
`ao_loaded`: When the script is loaded and ready for interaction.

`ao_event_sent`: Dispatched whenever an event has been sent to the CDP.
The event data is the response from the CDP.

`ao_recognized`: When a profiled has been recognized and data is available in frontend. The event data is the profile fetched from the CDP.

## Utility object

Window.argo is an instance of the Arrigoo utility class that communicates with the CDP. All requests will have a profile and session ID as well as current URL.

The most important methods of the class are:

`sendInitEvent()`
This is intended as being the initialization of a pageview from the CDP perspective. If nothing is overridden (more about that later), it will send a pageview event to the CDP.
If utm_source is set, it will be used for the source parameter.
If a gclid is set in the query, the source will be set to ‘ad’.
Otherwise, the source will be ‘web’.

If a utm_campaign is specified in the query, the value will be used for the string value on the pageview. Otherwise, the string value will be empty.

`send(event: string, strval?: string, eventOverrides?: EventInterface)`
For sending other events such as download, login, newsletter signup etc. the send method can be used.
The first parameter is the event type. Remember that the event type must be registered in the CDP before it can be received.
The second parameter is the string value of the event.
The third parameter allows for overriding all attributes on the event. See interface below.

`getFullProfile()`
Returns the full profile as JSON.

`set(key: string, value: any)`
Override a default value on the initial event. See specifications for keys and value type in the interface definition. Useful for setting e.g. topics on the pageview event:

```javascript
 window.argo.set(‘topics’, [‘news’, ‘table tennis’])
```

Interfaces
The primary interfaces to know are for the profile and the event.

```javascript
export interface EventI {
   cid: string; // Profile/Customer ID
   sid: number; // Session ID
   evt: string; // Event type must match a specified event type.
   topics?: string[]; // Event topics/tags
   strval: string; // Event string value
   intval: number; // Event integer value
   src: string; // 
   url: string; // Current URL
   fv?: boolean; // First visit
   ns?: boolean; // New session
   ref?: string; // Referrer
   ident?: IdentifierValueI; // Identifier to look up a user instantly.
}


export interface ProfileI {
   cid: string;
   s: string[]; // Segments - system titles.
   p: Record<string, any>; // Properties. Keyed by system title.
   i: IdentifierValueI[]; // Identifiers.
   c: ConsentI[]; // Profile consents.
}


interface IdentifierValueI {
   id_type: 'email' | 'phone' | 'foreignid1' | 'foreignid2' | 'foreignid3';
   id_value: string;
}
export interface ConsentI {
   cid?: string;
   scope: string;
   type: string;
   status: 'GRANT' | 'DENY' | 'REVOKE';
   timestamp?: string;
}
```