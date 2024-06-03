![image](https://github.com/bernard-dali/cloudflare-bypass-python-demo/assets/151824231/c6f6dcd7-99a6-4eaf-b50f-ac6af138ca97)

# Example of solving Cloudflare Challenge page using pyppeteer (python)

This example demonstrates how to bypass the Cloudflare Challenge page captcha located on https://2captcha.com/demo/cloudflare-turnstile-challenge using python. The [pyppeteer](https://pypi.org/project/pyppeteer/) tool is used for automation. The captcha is solved in the [2captcha](https://2captcha.com/?from=22771395) service.

Cloudflare Challenge page - is a captcha that is displayed as a separate page, preventing the site from opening. Bypassing this captcha requires an unusual approach. The solution is to inject code into the page before it loads, this example uses `page.evaluateOnNewDocument` provided by pyppeteer. Then the embedded code intercepts the parameters of the found turnstile captcha, the captcha is sent to [2captcha](https://2captcha.com/?from=22771395), after receiving the result the answer is applied on the page.

The idea and some of the code is based on this repository https://github.com/2captcha/cloudflare-demo.
You can read more about bypassing this type of captcha in the article [Bypassing Cloudflare Challenge with Puppeteer and 2Captcha](https://2captcha.com/blog/bypassing-cloudflare-challenge-with-puppeteer-and-2captcha?from=22771395). 

This example can be used to bypass most Cloudflare Challenge page captchas. To use it, just replace the URL with yours. Also in some cases you may encounter additional protection methods, such as Cloudflare blocking, in such cases you should first find out how to bypass Cloudflare blocking and then solve the captcha.

## Usage

1. Clone repo:

   `git clone https://github.com/bernard-dali/cloudflare-bypass-python-demo.git`

2. Install dependencies:

   `pip install -r requirements.txt`

3. Set apikey in `main.py`:

   `apikey = "2captcha api key"`

4. Run:

   `python main.py`

## Source code:
Python source code:
```python
import asyncio
import os
import json
PYPPETEER_CHROMIUM_REVISION = '1263111'
os.environ['PYPPETEER_CHROMIUM_REVISION'] = PYPPETEER_CHROMIUM_REVISION
from pyppeteer import launch
import time
import requests
apikey = "2captcha api key"

async def main():
    browser = await launch(headless=False, devtools=True, autoClose=False, args=['--no-sandbox', '--disable-setuid-sandbox'])
    page = await browser.newPage()

    # Executing the javascript code on the page
    await page.evaluateOnNewDocument(
        """
        () => {
          console.clear = () => console.log('Console was cleared')
          const i = setInterval(() => {
              if (window.turnstile) {
                  clearInterval(i)
                  window.turnstile.render = (a, b) => {
                      let params = {
                          sitekey: b.sitekey,
                          pageurl: window.location.href,
                          data: b.cData,
                          pagedata: b.chlPageData,
                          action: b.action,
                          userAgent: navigator.userAgent,
                          json: 1
                      }
                      // we will intercept the message in puppeeter
                      console.log('intercepted-params:' + JSON.stringify(params))
                      window.cfCallback = b.callback
                      return
                  }
              }
          }, 50)
        }
        """
    )    

    # Intercept console messages to catch a message containing 'intercepted-params:'
    async def console_message_handler(msg):
        print(f"Dialog message: {msg}")
        txt = msg.text
        print(txt)
        if 'intercepted-params:' in txt:
            params = json.loads(txt.replace('intercepted-params:', ''))
            print(params)
            try:
                # Captcha params
                payload = {
                    "key": apikey,
                    "method": "turnstile",
                    "sitekey": params["sitekey"],
                    "pageurl": params["pageurl"],
                    "data": params["data"],
                    "pagedata": params["pagedata"],
                    "action": params["action"],
                    "useragent": params["userAgent"],
                    "json": 1,
                    }
                # Send captcha to 2captcha
                response = requests.post(f"https://2captcha.com/in.php?", data=payload)
                print("Captcha sent")
                print(response.text)
                captcha_id = response.json()["request"]
                time.sleep(2)
                # Getting a captcha response
                while True:
                    solution = requests.get(f"https://2captcha.com/res.php?key={apikey}&action=get&json=1&id={captcha_id}").json()
                    if solution["request"] == "CAPCHA_NOT_READY":
                        print(solution["request"])
                        time.sleep(1)
                    elif "ERROR" in solution["request"]:
                        print(solution["request"])
                    else:
                        print(solution)
                        break

                # Use the received captcha response. Pass the answer to the configured callback function `cfCallback`
                await page.evaluate('cfCallback', solution["request"])
            except Exception as e:
                print(e)
                await browser.close()
        else:
            return

    # Watch the console
    page.on('console', lambda msg: asyncio.ensure_future(console_message_handler(msg)))
    # Open target page
    # Just change this url to the target page you want
    await page.goto('https://2captcha.com/demo/cloudflare-turnstile-challenge')

# Create an asyncio event loop and run function main()
loop = asyncio.get_event_loop()
loop.run_until_complete(main())

```
