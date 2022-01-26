# g18n

## [This is an early draft on my existing code for a future npm package. It works but it's better not to use it until proper release]


I don't like how i18n libs work in JS. While they make sense when you don't want to have all the languages texts loaded at the same time,
there are some cases where having all of them is useful, like in React Native Apps.

Also, creating the dictionaries with the i18n libs may be an issue if you type wrongly some text key or if you forget to translate some texts.

With this library and with TS, it won't allow missing or wrong texts keys. It also allows having functions for the texts.

Instead of the `language: {textId: translation}` philosophy used by the famous libs, it uses `textId: {language: {translation}` pattern.

It aims to be easy and fast for the developer to use. Uses proxies to allow its programagical working.

## Usage:

### Translations setup
```ts
const myResource = createResource(['en', 'pt'], {
  // Simple translations
  loginRequired: {
    en: 'You need to be logged to execute this action.',
    pt: 'Você precisa estar logado para executar esta ação.',
  },
  // Groups
  _license: {
    pressToBuy: {
      en: 'Press here to adquire your license!',
      pt: 'Pressione aqui para adquirir sua licença!',
    },
    // Functions. The type of the args are TS enforced.
    paused: (date: string) => ({
      en: `The license is paused until ${date}, as requested!`,
      pt: `A licença está pausada até ${date}, conforme solicitado!`,
    }) as const,
  }
} as const)
```

### Usage

```tsx
// You can use just T when not using React, but it won't automatically rerender on language change.
export const { useT, T, language, languages } = createT({
  resource: myResource,
  languages: ['en', 'pt'],
  initialLanguage: 'pt',
});

export function Component(): JSX.Element {
  const { T } = useT();
  if (!logged)
    return <Text>{T.loginRequired}</Text>
  else
    // TS typesafe argument
    return <Text>{T._license.paused('January 25, 2022')}</Text>
}
```

As we use `as const`, you may see the translation texts when hovering the T.textId.

<details><summary>Code</summary>

```ts
import { createGlobalState } from 'react-hooks-global-state';
import { Id } from '../utils/utils';



const defaultLanguage = 'en';
export let fallbackLanguage = defaultLanguage;
let languages: ReadonlyArray<string> = [fallbackLanguage];
let resource: Resource = {};
let language: string = '';
let T: any = undefined;
let baseNodesProxies: NodeProxy = {};
let onLanguageChange: ((language: string) => void) | undefined = undefined;

/** If calledCreated was already called */
let calledCreateT = false;
/** If {awaitInitT: true} prop was passed in createT */
let usingInitT = false;
/** If createT was already called (and also initT, if configured) */
let created = false;



type MyGlobalState = {
  language: string,
  T: any;
};
// From react-hooks-global-state/dist/src/createGlobalState.d.ts
type UseGlobalState<State> = <StateKey extends keyof State>(stateKey: StateKey) => readonly [State[StateKey], (u: import('react').SetStateAction<State[StateKey]>) => void];
type SetGlobalState<State> = <StateKey_2 extends keyof State>(stateKey: StateKey_2, update: import('react').SetStateAction<State[StateKey_2]>) => void;
let useGlobalState: UseGlobalState<MyGlobalState> | undefined = undefined;
let setGlobalState: SetGlobalState<MyGlobalState> | undefined = undefined;




export type Resource<L extends string = string> = {
  [id: string]: {
    [language in L]: string;
  }
    | ((...args: any[]) => {[language in L]: string})
    | Resource<L>
}



type InitTFunParams = {
  language?: string;
}
export let initT: (args?: InitTFunParams) => void = () => {
  if (!calledCreateT)
    throw new Error('initT called but createT wasn\'t called before.');
  if (!usingInitT)
    throw new Error('initT called but awaitInitT prop wasn\'t passed in createT');
};



type I18nParam<L extends string, R extends Resource> = Readonly<{
  languages?: ReadonlyArray<L>,
  initialLanguage?: L,
  resource: R,
  onLanguageChange?: (language: string) => void;
  fallbackLanguage?: string;
  /** If true, the createT will only take effect when initT() function is called.
   * @default false */
  awaitInitT?: boolean;
  // strictCheck?: boolean // check for extra translations and missing languages on init.
  // onMissingLanguage?: (textId, language) => void, function called on missing translation and T used.
  // showTranslationsTypes // if it will show the translations in intelissense when hovering T.[x].

}>


export function createT<L extends string, R extends Resource>({
  awaitInitT = false,
  initialLanguage = 'en' as any,
  languages: languagesProp = ['en'] as any,
  resource: resourceProp,
  onLanguageChange: onLanguageChangeProp,
  fallbackLanguage: fallbackLanguageProp = defaultLanguage,
}: I18nParam<L, R>) {

  calledCreateT = true;
  const fun = (args?: InitTFunParams) => {
    // if (created)
    //   console.warn('initT was called again but createT has been already successfully and fully executed.');
    fallbackLanguage = fallbackLanguageProp;
    // Try first to use current language, if Fast Refresh,
    // then check initT language argument, finally use createT language arg.
    language = language || args?.language || initialLanguage;
    resource = resourceProp;
    onLanguageChange = onLanguageChangeProp;
    languages = languagesProp;
    baseNodesProxies = {}; // Reset it.
    T = createProxy({ resource, subNodesProxies: baseNodesProxies });
    const createdGlobalState = createGlobalState<MyGlobalState>({ language, T });
    useGlobalState = createdGlobalState.useGlobalState;
    setGlobalState = createdGlobalState.setGlobalState;
    created = true;
  };
  if (awaitInitT && !created) {
    // Fast refresh workaround.
    initT = (args?: InitTFunParams) => {
      fun(args);
      initT = () => null;
    };
    usingInitT = true;
  } else // If not awaiting or recreating (like Fast Refresh)
    fun();

  return {
    useT: () => useTInternal<L, ParseR<R>>(),
    // Getters to keep it updated.
    /** Translator */
    get T() {
      return T as unknown as ParseR<R>;
    },
    /** Current language */
    get language() {
      return language as L;
    },
    /** Available languages */
    get languages() {
      return languages as L[];
    },
  };
}



function isSubNode(key: string) {
  return key[0] === '_';
}


type NodeProxy = Record<string, any>



// Outside proxy, so it won't be created for each node.
const proxyGet = (
  { resource, subNodesProxies, selectionId: selectedId }:
  {resource: Resource<string>, selectionId: string, subNodesProxies: NodeProxy},
): any => {

  if (['$$typeof', 'prototype'].includes(selectedId)) // Those textIds may happen on Fast Refresh in RN, for some unknown reason.
    return 'TYPE_OF_ERROR'; // it shouldnt appear anywhere, but if it do, we can track it down here.

  // console.log('l, r, t', language, resource, textId);

  const selectedNode = resource[selectedId];

  if (!selectedNode) {
    const id = selectedId.substr(0, 30);
    console.warn(`Translations not found. TextId=${id} (name may have been shortened)`);
    return id;
  }


  if (typeof selectedNode === 'function')
    return (...args: any) => selectedNode(...args)[language];

  if (isSubNode(selectedId)) {
    if (!subNodesProxies[selectedId]) {
      subNodesProxies[selectedId] = createProxy({ resource: selectedNode as Resource<string>, subNodesProxies: {} });
    }

    return subNodesProxies[selectedId];
  }

  else // Is a simple translation node
    return selectedNode[language];
};



function createProxy({ subNodesProxies, resource }: {resource: Resource, subNodesProxies: NodeProxy}) {
  return new Proxy(resource, {
    get: (resource, selectionId: string) => proxyGet({ subNodesProxies, resource, selectionId }),
  }) as any as Record<string, string>;
}



// TODO overload so languages is optional
// TODO add some magic so only desired lang is processed in functional translations
/** @param languages - TS helper */
export function createResource<R extends Resource<L>, L extends string>(languages: ReadonlyArray<L>, myResource: R): R {
  return myResource;
}



export function setLanguage(newLanguage: string) {
  if (newLanguage !== language) {
    language = newLanguage;
    setGlobalState?.('language', newLanguage);
    T = createProxy({ resource, subNodesProxies: baseNodesProxies }); // We have to recreate the proxy to trigger React dep list. It will only recreate the top level resource proxy (T).
    setGlobalState?.('T', T);
    onLanguageChange?.(newLanguage);
  }
}



type ParseR<R extends Resource> = {[K in keyof R]: R[K] extends (args: any) => any ? ParseFun<R[K]> : ParseObj<R[K]>}
type ParseObj<T extends Record<string, any>> = T[keyof T] extends string ? Id<T[keyof T]> : Id<ParseR<T>> // Union vals if string
type ParseFun<T extends (args: any) => Record<string, any>> =
  T extends (...args: infer A) => infer R ? (...args: A) => ParseObj<R> : never

// type UseTRtn<L extends string, R extends Resource<L>> = {
//   language: L,
//   setLanguage: (language: string) => void;
//   T: R
// }
/** Hook */



function useTInternal<L extends string, T>() {
  if (!useGlobalState)
    throw new Error ('use18 Error: create18 wasn\'t called!');

  const [internalLanguage] = useGlobalState('language');
  const [internalT] = useGlobalState('T'); // Using it in a state so T can be used as dep in onEffect etc.

  return {
    language: internalLanguage as L, // not using internal so dev uses the same language value (on hook and outside)
    languages: languages as L[],
    setLanguage: setLanguage,
    T: internalT as unknown as T,
  };
}
```

</details>
