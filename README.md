# Ecourts India -data-extraction
#Here I have written an algorithm to extract case level time series data from ecourts website of governemt of India
##Code
import requests

from io import BytesIO
from PIL import Image
import os
import re
import subprocess
import tempfile
import json
import csv
import pandas as pd
from bs4 import BeautifulSoup
import time
from datetime import datetime
from requests.exceptions import ConnectionError
from urllib3.exceptions import ProtocolError
import multiprocessing


captcha_url = "https://services.ecourts.gov.in/ecourtindia_v4_bilingual/securimage/securimage_show.php?0.8580921212662784"
act_url = "https://services.ecourts.gov.in/ecourtindia_v4_bilingual/cases/s_actwise_qry.php"
final_url = "https://services.ecourts.gov.in/ecourtindia_v4_bilingual/cases/s_actwise_qry.php"
view_url = "https://services.ecourts.gov.in/ecourtindia_v4_bilingual/cases/o_civil_case_history.php"


# print(court_data.head())
# hist_table = pd.DataFrame({})

file = open('C:/Users/babu/Dropbox (Personal)/BMGF/babu_analysis/output_data/court_data_output_1.csv', 'a', newline='')

# identifying header
header = ['state_code', 'dist_code', 'court_code', 'act_code', 'case_type', 'filling', 'registration', 'cnr', 'first_hear_date', 'next_hear_date', 'stage', 'court_no_judge',
          'petitioner_advocate', 'respondent_advocate', 'under_act', 'under_sec', 'police_station', 'firno', 'year']

writer = csv.DictWriter(file, fieldnames=header)
# writing data row-wise into the csv file
writer.writeheader()

file2 = open('C:/Users/babu/Dropbox (Personal)/BMGF/babu_analysis/history_data/case_history_1f.csv', 'a', newline='')
ch_header = ['rg_no', 'Judge', 'bod', 'hd', 'poh']

ch_writer = csv.DictWriter(file2, fieldnames=ch_header)
# writing data row-wise into the csv file
ch_writer.writeheader()

class ActDataConsolidation(object):

    def read_captcha(self):
        readpath = 'C:/Users/babu/Dropbox (Personal)/BMGF/babu_analysis/captcha.png'
        savepath = 'C:/Users/babu/Dropbox (Personal)/BMGF/babu_analysis/'
        payload = {}
        headers = {
            'Connection': 'keep-alive',
            'sec-ch-ua': '"Google Chrome";v="87", " Not;A Brand";v="99", "Chromium";v="87"',
            'sec-ch-ua-mobile': '?0',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36',
            'Accept': 'image/avif,image/webp,image/apng,image/*,*/*;q=0.8',
            'Sec-Fetch-Site': 'same-origin',
            'Sec-Fetch-Mode': 'no-cors',
            'Sec-Fetch-Dest': 'image',
            'Referer': 'https://services.ecourts.gov.in/ecourtindia_v4_bilingual/cases/s_actwise.php?state=D&state_cd=18&dist_cd=11',
            'Accept-Language': 'en-US,en;q=0.9',
            'Cookie': 'PHPSESSID=ss56ec7a9ti2hcqef7kleu1pqn; PHPSESSID=og1b75tea7og3105995v6vm2rd'
        }
        response = requests.request("GET", captcha_url, headers=headers, data=payload)
        image = Image.open(BytesIO(response.content))
        image.save(savepath + 'captcha.png')
        data_img = self.threshold(readpath)
        value = self.tesseract(data_img)
        return value

    def get_act_data(self, state_code, dist_code, court_code):
        payload = 'court_code={}&lang=&action_code=fillActType&search_act=&state_code={}&dist_code={}'.format(
            court_code, state_code, dist_code)
        headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Cookie': 'PHPSESSID=og1b75tea7og3105995v6vm2rd'
        }
        response = requests.request("POST", act_url, headers=headers, data=payload)
        # print(response.content)
        final_data = []
        data = response.text.split("#")
        for each in data:
            if each:
                _list = each.split("~")
                final_data.append([_list[0], _list[1]])

        # with open('/Users/babu/Desktop/work/act_code_data.csv', mode='w', newline='') as csv_file:
        #     fieldnames = ['act_code', 'act_name']
        #     #writer = csv.DictWriter(csv_file, fieldnames=fieldnames)
        #     writer = csv.writer(csv_file, delimiter=',', quotechar='"', quoting=csv.QUOTE_MINIMAL)
        #     #writer.writeheader()
        #     for i, _data in enumerate(final_data):
        #         if i != 0:
        #             # import pdb; pdb.set_trace()
        #             writer.writerow(_data)
        return final_data

    def get_final_data(self, state_code, dist_code, actcode, court_code):
        captcha = self.read_captcha()
        #print ("retrieved captcha {}".format(captcha))
        payload = 'court_code={}&state_code={}&dist_code={}&search_act=&actcode={}&f=Pending&under_sec=&action_code=showRecords&captcha={}&lang='.format(
            court_code, state_code, dist_code, actcode, captcha)
        headers = {
            'Connection': 'keep-alive',
            'sec-ch-ua': '"Google Chrome";v="87", " Not;A Brand";v="99", "Chromium";v="87"',
            'Accept': 'application/json, text/javascript, */*; q=0.01',
            'X-Requested-With': 'XMLHttpRequest',
            'sec-ch-ua-mobile': '?0',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36',
            'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
            'Origin': 'https://services.ecourts.gov.in',
            'Sec-Fetch-Site': 'same-origin',
            'Sec-Fetch-Mode': 'cors',
            'Sec-Fetch-Dest': 'empty',
            'Referer': 'https://services.ecourts.gov.in/ecourtindia_v4_bilingual/cases/s_actwise.php?state=D&state_cd=18&dist_cd=11',
            'Accept-Language': 'en-US,en;q=0.9',
            'Cookie': 'PHPSESSID=ss56ec7a9ti2hcqef7kleu1pqn; PHPSESSID=og1b75tea7og3105995v6vm2rd'
        }

        response = requests.request("POST", final_url, headers=headers, data=payload)
        data = json.loads(response.content)
        #import pdb; pdb.set_trace()
        if not data.get('con'):
            pass
            #print("con cehck of data {}".format(data))
        else:
            if data and data['con'] == 'Invalid Captcha':
                #print ("captcha is not valid, response {}".format(data))
                self.get_final_data(state_code, dist_code, actcode, court_code)
            else:
                pass
                #import pdb; pdb.set_trace()
                #print ("\n\n\n\ndata {}".format(data))
                if not data.get('con'):
                    pass
                    #print("con cehck of data {}".format(data))
                else:
                    if data and data['con'][0] != None:
                        refined_data = json.loads(data['con'][0])
                        for each in refined_data:
                            cino = each['cino']
                            case_no = each['case_no']
                            self.read_view(state_code, dist_code, court_code, cino, case_no, actcode)

    def read_view(self, state_code, dist_code, court_code, cino, case_no, actcode):
        #import ipdb; ipdb.set_trace()
        payload = 'state_code={}&dist_code={}&court_code={}&cino={}&case_no={}'.format(
            state_code, dist_code, court_code, cino, case_no)
        headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'Accept': '*/*'
        }
        response = requests.request("POST", view_url, headers=headers, data=payload)
        self.read_html(response.text, state_code, dist_code, court_code, actcode)
        self.read_pd_html(response.text)

    def parse_captcha(self, filename):
        """Return the text for thie image using Tesseract
        """
        img = self.threshold(filename)
        return self.tesseract(img)

    def threshold(self, filename, limit=100):
        """Make text more clear by thresholding all pixels above / below this limit to white / black
        """
        # read in colour channels
        img = Image.open(filename)
        # resize to make more clearer
        m = 1.5
        img = img.resize((int(img.size[0] * m), int(img.size[1] * m))).convert('RGBA')
        pixdata = img.load()

        for y in range(img.size[1]):
            for x in range(img.size[0]):
                if pixdata[x, y][0] < limit:
                    # make dark color black
                    pixdata[x, y] = (0, 0, 0, 255)
                else:
                    # make light color white
                    pixdata[x, y] = (255, 255, 255, 255)
        img.save(filename)
        return img.convert('L')  # convert image to single channel greyscale

    def call_command(self, *args):
        """call given command arguments, raise exception if error, and return output
        """
        c = subprocess.Popen(args, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = c.communicate()
        if c.returncode != 0:
            if error:
                print (error)
            print ("Error running `%s'" % ' '.join(args))
        return output

    def tesseract(self, image):
        """Decode image with Tesseract  
        """
        # create temporary file for tiff image required as input to tesseract
        input_file = tempfile.NamedTemporaryFile(suffix='.tif')
        image.save(input_file.name)

        # perform OCR
        output_filename = input_file.name.replace('.tif', '.txt')
        self.call_command('tesseract', input_file.name, output_filename.replace('.txt', ''))

        # read in result from output file
        result = open(output_filename).read()
        os.remove(output_filename)
        return self.clean(result)


    def gocr(self, image):
        """Decode image with gocr
        """
        input_file = tempfile.NamedTemporaryFile(suffix='.ppm')
        image.save(input_file.name)
        result = self.call_command('gocr', '-i', input_file.name)
        return self.clean(result)

    def ocrad(self, image):
        """Decode image with ocrad
        """
        input_file = tempfile.NamedTemporaryFile(suffix='.ppm')
        image.save(input_file.name)
        result = self.call_command('ocrad', input_file.name)
        return self.clean(result)

    def clean(self, s):
        """Standardize the OCR output
        """
        # remove non-alpha numeric text
        return re.sub('[\W]', '', s)

    def read_html(self, html, state_code, dist_code, court_code, actcode):
        list_header = []
        soup = BeautifulSoup(html, 'html.parser')

        #p1=soup.find_all('span', class_='case_details_table')

        p1_result = [p1.text.strip()
                     for p1 in soup.find_all('span', 'case_details_table')]

        p12_before = soup.select_one('div[style="width:700px;margin-top:0px;background-color:#FBF6D9;font-size:1em ;font-weight:200;color:red;text-align:center;"]');
        if p12_before:
            p12_result = [p12.text.strip() for p12 in p12_before]
        else:
            p12_result = 'Null'

        p2_result = [p2.text.strip()
                     for p2 in soup.find_all('span', class_='Petitioner_Advocate_table')]
        p3_result = [p3.text.strip()
                     for p3 in soup.find_all('span', class_='Respondent_Advocate_table')]
        p4 = soup.find('table', id='act_table')
        p4_result = None
        if p4:
            rows = p4.findAll('tr')
            p4_result = [[td.findChildren(text=True) for td in tr.findAll("td")] for tr in rows]

        p5_result = [p5.text.strip()
                     for p5 in soup.find_all('span', class_='FIR_details_table')]
        # Save in a CSV File
        writer.writerow({'state_code': state_code,
                         'dist_code': dist_code,
                         'court_code': court_code,
                         'act_code': actcode,
                         'case_type': p1_result[0],
                         'filling': p1_result[1],
                         'registration': p1_result[2],
                         'cnr': p1_result[3],
                         'first_hear_date': p12_result[0],
                         'next_hear_date': p12_result[2] if p12_result else "Null",
                         'stage': p12_result[4],
                         'court_no_judge': p12_result[6],
                         'petitioner_advocate': p2_result[0],
                         'respondent_advocate': p3_result[0],
                         'under_act': p4_result[1] if p4_result else "Null",
                         'police_station': p5_result[0] if p5_result else "Null"

                         })

    def read_pd_html(self, html):
        # import ipdb
        # ipdb.set_trace()
        tables = pd.read_html(html)
        try:
            df = tables[2]
            if not df.empty:
                rg_no = df.get('Registration Number', pd.DataFrame({}))
                if not rg_no.empty:
                    Judge = df['Judge']
                    bod = df['Business On Date']
                    hd = df['Hearing Date']
                    poh = df['Purpose of hearing']
                    n = len(df['Registration Number'])
                    for i in range(n):
                        ch_writer.writerow({
                            'rg_no': "babu_{}".format(rg_no[i]),
                            'Judge': Judge[i],
                            'bod': bod[i],
                            'hd': hd[i],
                            'poh': poh[i]
                        })
        except IndexError:
            pass


court_data = pd.read_csv('C:/Users/babu/Dropbox (Personal)/BMGF/babu_analysis/courtroom_url.csv')


def final_run(state_code, dist_code):
    c1 = court_data.loc[(court_data.state_code == state_code) & (court_data.dist_code == dist_code), 'court_code']
    _list = c1.tolist()
    exclude_court_code = []
    for court_code in _list:
        if court_code not in exclude_court_code:
            act_data = ActDataConsolidation()
            # act_data.read_captcha()
            act_final_data = act_data.get_act_data(state_code, dist_code, court_code)
            for each in act_final_data:
                try:
                    actcode = each[0]
                    start_date = datetime.now()
                    print ("\n\n\n\n\n\n\n retrieving data for act {}, state_code {}, court_code {}, dist_code {}".format(
                        actcode, state_code, court_code, dist_code))
                    act_data.get_final_data(state_code, dist_code, actcode, court_code)
                    end_date = datetime.now()
                    print ("Time taken for processing of act {}, state_code {}, court_code {}, dist_code {} is {} seconds".format(
                        actcode, state_code, court_code, dist_code, (end_date - start_date).seconds))
                except (ConnectionError, ProtocolError):
                    actcode = each[0]
                    start_date = datetime.now()
                    print ("\n\n\n\n\n\n\n retrieving data for act {}, state_code {}, court_code {}, dist_code {}".format(
                        actcode, state_code, court_code, dist_code))
                    act_data.get_final_data(state_code, dist_code, actcode, court_code)
                    end_date = datetime.now()
                    print ("Time taken for processing of act {}, state_code {}, court_code {}, dist_code {} is {} seconds".format(
                        actcode, state_code, court_code, dist_code, (end_date - start_date).seconds))


if __name__ == "__main__":
    # creating processes
    start_date = datetime.now()
    p1 = multiprocessing.Process(target=final_run, args=(2, 3))
    # p2 = multiprocessing.Process(target=final_run, args=(2, 4))
    # p3 = multiprocessing.Process(target=final_run, args=(2, 5))
    # p4 = multiprocessing.Process(target=final_run, args=(2, 7))
    p1.start()
    # p2.start()
    # p3.start()
    # p4.start()
    # p1.join()
    # p2.join()
    # p3.join()
    p1.join()
    end_date = datetime.now()
    # both processes finished
    print("Done!, Total time taken in this process is {} minutes".format((end_date - start_date).seconds/60))

