<h1 align="center">

🌀 Portal-Moei

</h1>

## Live instances

- Development: https://portal-moei.vercel.app/
- Production: https://opendata.moei.gov.ae/

## Getting Started

### Setup

Install a recent version of Node. We recommend using Node v18.

```bash
npm i # install dependencies
npm run dev # starts dev mode
```

Open [http://localhost:3000](http://localhost:3000) to see the home page 🎉

You can start editing the page by modifying `/pages/index.tsx`. The page auto-updates as you edit the file.

To connect your frontend to the backend (DMS and CMS), you will need to setup the following environment variables:

```bash
export DMS=http://ckan-domain.com
export CMS=http://ghost-cms-domain.com # Note that we're using Ghost CMS for this frontend
export CMS_KEY=your-content-key # You can get it from Ghost CMS settings
```

## Guide

### Styling 🎨

We use Tailwind as a CSS framework. Take a look at `/styles/index.css` to see what we're importing from Tailwind bundle. You can also configure Tailwind using `tailwind.config.js` file.

Have a look at Next.js support of CSS and ways of writing CSS:

https://nextjs.org/docs/basic-features/built-in-css-support

### Backend

So far the app is running with mocked data behind. You can connect CMS and DMS backends easily via environment variables:

```console
$ export DMS=http://ckan:5000
$ export CMS=http://ghost-cms.domain
```

> Note that we don't yet have implementations for the following CKAN features:
>
> - Auth

### Routes

These are the default routes set up in the "starter" app.

- Home `/`
- Search `/search`
- Dataset `/@org/dataset`
- Resource `/@org/dataset/r/resource`
- Topics (aka group in CKAN or categories here) `/topic`
- Static pages, eg, `/about` etc. from CMS or can do it without external CMS, e.g., in Next.js

### New Routes

In this project, there are also some new routes:

- National Open Data `/national-open-data`
- Geo Data `/geo-data`
- Realtime Data `/realtime-data`

### Data fetching

We use Apollo client which allows us to query data with GraphQL.
Note that we don't have Apollo Server but we connect CKAN API using [`apollo-link-rest`](https://www.apollographql.com/docs/link/links/rest/) module. You can see how it works in [lib/apolloClient.ts](https://github.com/datopian/portal-moei/blob/master/lib/apolloClient.ts) and then have a look at [pages/\_app.tsx](https://github.com/datopian/portal-moei/blob/master/pages/_app.tsx).

For development/debugging purposes, we suggest installing the Chrome extension - https://chrome.google.com/webstore/detail/apollo-client-developer-t/jdkknkkbebbapilgoeccciglkfbmbnfm.

#### i18n configuration

Portal.js is configured by default to support multiple subpaths for language translation. In this specific project `English` and `Arabic` are supported. When switching to `Arabic`, this portal also changes all the content flow from `LTR` to `RTL`. But for subsequent users, this following steps can be used to configure i18n for other languages;

1.  Update `i18n.json`, to add more languages to the i18n locales

```js
{
  "locales": ["en", "ar", "pt-br"], // Add the new language code here
  "defaultLocale": "en",  // Set the default language
  "pages": {
    "*": ["common"]
  }
}
```

2. Create a folder for the language in `locales` using the language code as the name. E.g. `locales/pt-br`.

3. In the language folder, different namespace files (json) can be created for each translation. For the `index.js` use-case, I named it `common.json`

```json
// locales/en/common.json
{
   "title" : "Portal js in English",
}

// locales/ar/common.json
{
   "title" : "Portal js in Arabic",
}
```

4. To use on pages using Server-side Props.

```js
import { loadNamespaces } from './_app';
import useTranslation from 'next-translate/useTranslation';

const Home: React.FC = ()=> {
  const { t } = useTranslation('common');
  return (
    <div>{t(`title`)}</div> // we use title based on the common.json data
  );
};

export const getServerSideProps: GetServerSideProps = async ({ locale }) => {
      ........  ........
  return {
    props : {
      _ns:  await loadNamespaces(['common'], locale),
    }
  };
};

```

5. Go to the browser and view the changes using language subpath like this `http://localhost:3000` and `http://localhost:3000/ar`. **Note** The subpath also activates Chrome language translator. We also have Google Translate plugin that is enabled an available in the portal in the top right button: `Languages`

#### Pre-fetch data in the server-side

When visiting a dataset page, you may want to fetch the dataset metadata in the server-side. To do so, you can use `getServerSideProps` function from NextJS:

```javascript
import { GetServerSideProps } from 'next';
import { initializeApollo } from '../lib/apolloClient';
import gql from 'graphql-tag';

const QUERY = gql`
  query dataset($id: String) {
    dataset(id: $id) @rest(type: "Response", path: "package_show?{args}") {
      result
    }
  }
`;

...

export const getServerSideProps: GetServerSideProps = async (context) => {
  const apolloClient = initializeApollo();

  await apolloClient.query({
    query: QUERY,
    variables: {
      id: 'my-dataset'
    },
  });

  return {
    props: {
      initialApolloState: apolloClient.cache.extract(),
    },
  };
};
```

This would fetch the data from DMS and save it in the Apollo cache so that we can query it again from the components.

#### Access data from a component

Consider situation when rendering a component for org info on the dataset page. We already have pre-fetched dataset metadata that includes `organization` property with attributes such as `name`, `title` etc. We can now query only organization part for our `Org` component:

```javascript
import { useQuery } from '@apollo/react-hooks';
import gql from 'graphql-tag';

export const GET_ORG_QUERY = gql`
  query dataset($id: String) {
    dataset(id: $id) @rest(type: "Response", path: "package_show?{args}") {
      result {
        organization {
          name
          title
          image_url
        }
      }
    }
  }
`;

export default function Org({ variables }) {
  const { loading, error, data } = useQuery(
    GET_ORG_QUERY,
    {
      variables: { id: 'my-dataset' }
    }
  );

  ...

  const { organization } = data.dataset.result;

  return (
    <>
      {organization ? (
        <>
          <img
            src={
              organization.image_url
            }
            className="h-5 w-5 mr-2 inline-block"
          />
          <Link href={`/@${organization.name}`}>
            <div className="font-semibold text-primary underline">
              {organization.title || organization.name}
            </div>
          </Link>
        </>
      ) : (
        ''
      )}
    </>
  );
}
```

### CMS

This project uses [GhostCMS](https://ghost.org/) as a Content Management System. That's where pages and posts are managed, as well as other settings, such as the navbar and footer links.

#### Tags

Tags are a way of categorizing items. Whenever in GhostCMS you create an item, such as posts and pages, you are gonna see that there's always a field to associate tags to that item ([click here to read more about tags in GhostCMS](https://ghost.org/help/organising-content/)). Tags in this context also serve as an identifier for the intended post page. The following tags are needed for some special features in this application:

- `About`: items that populate the `/about` page.
- `Geo`: items that populate the `/geo-data` page.
- `Realtime`: items that populate the `realtime-data` page.

- `Arabic` correspondants: populate the arabic version of said pages.

#### Posts

Posts are being used to provide articles under the `About`, `Geo Data` and `Realtime Data` pages for the front end appication. Posts have bilingual support as mentioned above, the related recommendation is that when creating posts in Arabic to append the `Arabic` to the post's tags, in order to make it consistent application wide. So, for example, if there was a post about health status in the Realtime Data page, the tag for the English version could be `Realtime` and if this post was to be translated, the tag for the Arabic equivalent post would be `Realtime Arabic`.

#### Custom Snippets

GhostCMS supports snippets that work as custom widgets. These can be used in `Posts`.

### Tests

We use Jest for running tests:

```bash
npm run test # or npm run test

# turn on watching
npm run test --watch
```

We use Cypress tests as well

```
npm run e2e
```

## Contact and Request Dataset form Config

if using `gmail` service

```
MAIL_ACCOUNT= email adress
MAIL_PASSWORD= generated app password
CONTACT_EMAIL= contact email recipient
REQUEST_DATA_EMAIL = email recipient
```

using `smtp`, following should be added to the above

```
MAIL_PORT = smtp port e.g 2525
MAIL_SERVER = smtp server e.g smtp.mailtrap.io
```

### Architecture

- Language: Typescript / Javascript.
- Framework: NextJS - https://nextjs.org/.
- Data layer API: GraphQL using Apollo. So controllers access data using GraphQL “gatsby like”.

### Key Pages

See https://tech.datopian.com/frontend/.
