# 乱码还原

## 解题过程

Flag2 的解法同样适于 flag1，故只讨论 flag2。问题的关键其实是恢复 utf8 转 shift_jis 中丢掉的字节。

首先统计了佛语中( 不包括 “佛曰：” ) 可能出现的字节，发现都 e3 - e9 开头，查 utf8 的编码规范，后续的字节一定是 0x80-0xbf。

然后根据[这张表](https://encoding.spec.whatwg.org/shift_jis.html)，和运行脚本尝试，确定了 e3 - e9 这些字节一定不会丢。因为在 shift_jis 中， e3 - e9 可以作为双字节开头，也总可以作为双字节的第二个字节。

所以佛语的长度是可以确定的，我们只需要补上 utf8 编码中丢掉的第二第三个字节。对于缺失字节的，求解所有可能的原三字节是什么。下面的脚本在求解可能三字节时，尝试用状态机进行了一定剪枝，导致看起来极其臃肿，不确定是否必要。

由于反复 encode 导致内容很长，进而导致可能的结果数量爆炸，不能暴力遍历。但是，由于 AES 是分块加密算法，是可以先只解密一部分内容的，从前向后一段段解密的。这使复杂度从乘法变成加法。

至于如何分段，赶进度不想认真思考了，直接遍历长度去分段解密，然后看解密出的内容是否都是 ASCII，如果是则这段解密成功，再解密下一段

脚本有亿点点乱，，，

~~~python
from Crypto.Cipher import AES
from random import choice
from collections import defaultdict
from copy import deepcopy

KEY = b'XDXDtudou@KeyFansClub^_^Encode!!'
IV = b'Potato@Key@_@=_='

TUDOU = [
    '滅', '苦', '婆', '娑', '耶', '陀', '跋', '多', '漫', '都', '殿', '悉', '夜', '爍', '帝', '吉',
    '利', '阿', '無', '南', '那', '怛', '喝', '羯', '勝', '摩', '伽', '謹', '波', '者', '穆', '僧',
    '室', '藝', '尼', '瑟', '地', '彌', '菩', '提', '蘇', '醯', '盧', '呼', '舍', '佛', '參', '沙',
    '伊', '隸', '麼', '遮', '闍', '度', '蒙', '孕', '薩', '夷', '迦', '他', '姪', '豆', '特', '逝',
    '朋', '輸', '楞', '栗', '寫', '數', '曳', '諦', '羅', '曰', '咒', '即', '密', '若', '般', '故',
    '不', '實', '真', '訶', '切', '一', '除', '能', '等', '是', '上', '明', '大', '神', '知', '三',
    '藐', '耨', '得', '依', '諸', '世', '槃', '涅', '竟', '究', '想', '夢', '倒', '顛', '離', '遠',
    '怖', '恐', '有', '礙', '心', '所', '以', '亦', '智', '道', '。', '集', '盡', '死', '老', '至']

BYTEMARK = ['冥', '奢', '梵', '呐', '俱', '哆', '怯', '諳', '罰', '侄', '缽', '皤']

options = [c.encode() for c in TUDOU] + [c.encode() for c in BYTEMARK]
fst_options = list(set([c[0] for c in options]))
sec_options = list(set([c[1] for c in options]))
thd_options = list(set([c[2] for c in options]))
sorted(fst_options)
sorted(sec_options)
sorted(thd_options)

# print('First:')
# for n in fst_options:
#     print(hex(n)[2:])
# print('\nSecond:')
# for n in sec_options:
#     print(hex(n)[2:])
# print('\nThird:')
# for n in thd_options:
#     print(hex(n)[2:])


# e9 会多一种 81 的情况
# e_0_2_may_lost = defaultdict(set)
# for i in thd_options:
#     for j in fst_options:
#         to_decode = hex(i)[2:] + hex(j)[2:] + 'a1'
#         if bytes.fromhex(to_decode).decode('shift_jis', errors='ignore').encode('shift_jis')[0] != i:
#             e_0_2_may_lost[j].add(i) 
# print('e_0_2_may_lost')
# for k, v in e_0_2_may_lost.items():
#     print(f'{hex(k)[2:]}: {[hex(n)[2:] for n in v]}')
e_start_lost_last = [0x80, 0xa0, 0x83, 0x84, 0x85, 0x86, 0x87]
e_start_lost_last_e9 = e_start_lost_last + [0x81]
e_start_lost_last_blank = list(range(0x80, 0xa1))
not_e_start_lookup = defaultdict(set)
not_e_start_lookup_e9 = defaultdict(set)
not_e_start_lookup_blank = defaultdict(set)


for chr_bytes in options:
    chr_bytes = chr_bytes[1:]
    to_decode = chr_bytes + b'\xe8\x80'
    decoded = to_decode.decode('shift_jis', errors='ignore').encode('shift_jis')
    t = decoded.find(b'\xe8')
    if t != 2:
        not_e_start_lookup[decoded[:t]].add(chr_bytes)
    to_decode = chr_bytes + b'\xe9\x80'
    decoded = to_decode.decode('shift_jis', errors='ignore').encode('shift_jis')
    t = decoded.find(b'\xe9')
    if t != 2:
        not_e_start_lookup_e9[decoded[:t]].add(chr_bytes)
    decoded = chr_bytes.decode('shift_jis', errors='ignore').encode('shift_jis')
    if decoded != chr_bytes:
        not_e_start_lookup_blank[decoded].add(chr_bytes)
# print(not_e_start_lookup)
# print(not_e_start_lookup_e9)
# print(not_e_start_lookup_blank)


with open("flag2.enc", "r", encoding="utf-8") as f:
    x = f.read()

t = x.encode('shift_jis')[8:]
l = list()
while t:
    if 0xe3 <= t[0] <= 0xe9:
        l.append(t[:1])
        t = t[1:]
    else:
        l[-1] += t[:1]
        t = t[1:]

STATE_NEW_BYTE = 0
STATE_NEXT_BYTE = 1

flag = []
# \x9a
state = STATE_NEXT_BYTE
for i in range(len(l)):
    t = l[i]
    next_t = l[i + 1] if i != len(l) - 1 else None
    if len(t) == 3:
        flag.append(t)
        next_byte = t[1]
        third_byte = t[2]
        if state == STATE_NEW_BYTE:
            if third_byte < 0xA1:
                state = STATE_NEXT_BYTE
        elif state == STATE_NEXT_BYTE:
            if next_byte < 0xA1 or (next_byte >= 0xA1 and third_byte >= 0xA1):
                state = STATE_NEW_BYTE
    elif len(t) == 2:
        next_byte = t[1]
        if state == STATE_NEW_BYTE:
            if not next_t:
                possiblities = e_start_lost_last_blank
            elif next_t[0] == 0xe9:
                possiblities = e_start_lost_last_e9
            else:
                possiblities = e_start_lost_last
            possiblities = [t + bytes.fromhex(hex(i)[2:]) for i in possiblities]
            possiblities = [t for t in possiblities if t in options]
            if len(possiblities) > 1:
                flag.append(possiblities)
            else:
                flag.append(possiblities[0])
        else:
            if not next_t:
                possiblities = not_e_start_lookup_blank[t[1:]]
            elif next_t[0] == 0xe9:
                possiblities = not_e_start_lookup_e9[t[1:]]
            else:
                possiblities = not_e_start_lookup[t[1:]]
            possiblities = [t[:1] + i for i in possiblities if t[:1] + i in options]
            if len(possiblities) > 1:
                flag.append(possiblities)
            else:
                flag.append(possiblities[0])
            if next_byte >= 0xA1:
                state = STATE_NEW_BYTE
    elif len(t) == 1:
        assert state == STATE_NEXT_BYTE, 'Other wise the next byte should be OK'
        state = STATE_NEW_BYTE
        if not next_t:
            possiblities = not_e_start_lookup_blank[b'']
        elif next_t[0] == 0xe9:
            possiblities = not_e_start_lookup_e9[b'']
        else:
            possiblities = not_e_start_lookup[b'']
        possiblities = [t + i for i in possiblities if t + i in options]
        if len(possiblities) > 1:
            flag.append(possiblities)
        else:
            flag.append(possiblities[0])

cnt = 1
NO_MARK = 3
MARK = 4
NO_KNOWN = 5
state = NO_MARK
for i in range(len(flag)):
    t = flag[i]
    if isinstance(t, list):
        if state == MARK:
            flag[i] = [c for c in flag[i] if c.decode() in TUDOU]
            state = NO_MARK
        else:
            has_mark = False
            for t in flag[i]:
                if t.decode() in BYTEMARK:
                    state == NO_KNOWN
                else:
                    state == NO_MARK
        cnt *= len(flag[i])
    else:
        if t.decode() in TUDOU:
            state = NO_MARK
        else:
            state = MARK




def Encrypt(plaintext):
    # 1. Encode Plaintext in UTF-16 Little Endian
    data = plaintext.encode('utf-16le')
    # 2. Add Paddings (PKCS7)
    pads = (- len(data)) % 16
    data = data + bytes(pads * [pads])
    # 3. Use AES-256-CBC to Encrypt
    cryptor = AES.new(KEY, AES.MODE_CBC, IV)
    result = cryptor.encrypt(data)
    # 4. Encode and Add Header
    return '佛曰：' + ''.join([TUDOU[i] if i < 128 else choice(BYTEMARK) + TUDOU[i - 128] for i in result])


def Decrypt(ciphertext):
        # 1. Remove Header and Decode
        if ciphertext.startswith('佛曰：'):
            ciphertext = ciphertext[3:]
            data = b''
            i = 0
            while i < len(ciphertext):
                if ciphertext[i] in BYTEMARK:
                    i = i + 1
                    data = data + bytes([TUDOU.index(ciphertext[i]) + 128])
                else:
                    data = data + bytes([TUDOU.index(ciphertext[i])])
                i = i + 1
            # 2. Use AES-256-CBC to Decrypt
            cryptor = AES.new(KEY, AES.MODE_CBC, IV)
            result = cryptor.decrypt(data)
            # 3. Remove Paddings (PKCS7)
            flag = result[-1]
            if flag < 16 and result[-flag] == flag:
                result = result[:-flag]
            # 4. Decode Plaintext with UTF-16 Little Endian
            return result.decode('utf-16le')
        else:
            return ''

i = 1000
while i < len(flag):
    i += 1000
    for j in range(i - 1000, i):
        forks = [[]]
        for t in flag[:j]:
            if isinstance(t, list):
                new_forks = []
                for c in t[1:]:
                    new_list = deepcopy(forks)
                    for f in new_list:
                        f.append(c)
                    new_forks.extend(new_list)
                for f in forks:
                    f.append(t[0])
                forks.extend(new_forks)
            else:
                for f in forks:
                    f.append(t)


        # forks = ['佛曰：' + b''.join(fork).decode() for fork in forks]

        for fork in forks:
            try:
                t = Decrypt('佛曰：' + b''.join(fork).decode())
                if t.isascii():
                    flag = fork[:j] + flag[j:]
            except:
                pass

print(t)
~~~

解密完成后的解码部分就比较简单了。b64 可能能解码 b32 编码的东西，但 b32 大概率不能解码 b64 编码的东西。而且 a85 和 b85 也大概率不兼容。所以从后向前 try 解码就是了

~~~python
from base64 import *
decoders = [b16decode, b32decode, b64decode, a85decode, b85decode]
decoders.reverse()
t = b'G5YUON2SFRMSON2GHA4WIWTCGZKSYSC2HYTUKUCIHA3T4KJJIJUCOSLRFVZUELZFGQ7EUQSWFU6DYS2SG5VSIPTLI46VAJKUH47SMSSHHRPVEILPGMWVGLZPHAYE4MJVIRRCCMSMFU3SSL2NIBJD4KRKIFJVQX2XHAYF2ZTKIFHW4ZCTJB2EO23TFRLUWZZYGV2DIVJTG5YUQK2YI4RDI4RQGMSS4OLAIVBEUNJTGNAT4MTDG5WVYQCXIFHWOPBHGQQW2TBNG5JUAZLOHYSUWQZMG5VF6ZKDIE2ES4DSFROTMK27HRQE6RJYFV2SWKSMG5YT4PLAIZAFGX3LGEZVCLZWH5KGAOLUIFJVQN2THAYF4XJLIJUDEXKHHQSTCQCBGU7DKS2VIFJFWRCBG5YUOU2JIYSUEYTDIJTSGILSIUVTY32ZIBYGCVRFHAZHGWBRIVBVQUB7HJHFIKKRFZWUUJSGIA3WEKSVHAZXAJCRI4RDG3DTHFIFEKBPHBHFGMLGFUQXKJJRHAZW6XR4I5MHKOSHHIVUCYK6JBYWYYLOIE3CSVB7G5WXAZSRI5MGWLZSGMWF6PSJGU7U4KJBGY5FS33RG5YUOUZJIUUDWPB5GMWFMNJXGRADCQLCFYQS4M3OHAZHE2CFIFVS2RJIH47TQWJHG5KDU4ZQIFJVQJKMG5VF44LHIYSTSZDFHE3EKYJZHBGD4YCSGZ2C6U2JHA3UQ2SWF5HXCNBIJFLTO3S6I46COKJFIFICE5BIG5YUQZBYIZNW2YDCGRBXCI3NJA3U45JKGY7SYNJOG5VFOSJQHVAGG3LKGY3UORJXH5NVC2LAGZYVINC2G5VFIWRSHVOCCILTHFIC2Z25HVBF2YBYIFICEP3SG5WW6JCVFNOCU22SGFGGS4DEHNCUQOLTFU4FKJC4G5YT2RZUIJGGYVCGHURFC42PHNFEYPSFIFJFWTR4G5VHELROI4RDMJKVI5AE6LDLIMZXCOLVGMSS43KQG5YUOUS3FQ6WGPC4GEZFITSKH5XUKJDRGY6TYQLVG5YT4WJDJA5EWOJNJA6EKYKMHBHHCJTAGNBFIJZ6HA3T2TLRIJGGGVS2GY7VS2ZRHNETISBYGY7SYNDMG5VF44LHIUUDWPDUJFHS4SZRJA3GGWTBIFJVCWZUG5WW6JDNINSSG3REHAYVENJXHRBGUVSXGYQS4YCCG5VFIWRWHYREKKTVHA4VWVJWJAREWLJOFRYUGQSRHAZHE5BEHYRDYLLRGQQVWPCFGRATAXLDHVZXK42WHA3FGNLEIFHWONSWFZYUILS5IA6UKKLBFV2SUKBNHA3FIUSZIUUEKODKHUSDSKJ7HVBFWZCWIE2SYWB5HAZHE2BZI46U45K4GZNEOWJKGBHV2ODVGNEUMLZTG5VFOSRNIBJHG33GII2VKNTAI45UMLZOIA2SQU3JG5VHERZKIRRCEQJ2JFLVYLRPJBYTQSCPGY5FUI3QG5YUQLLFFZJHKZZFIVCUOX2CHBJD6QZNIBLTK2BDHAZSSJSXFY3VSXBRJA7GYRCZFVZDI3STIFJFG3LLG5WW64DUFU5F4QSTGBITSTRRIYVS44DRGZ2C6VBQG5YUQSDRFY3WEW2UHRDT25KJI45UMLBMFU7WS3JSG5WV2XTDHQUEGSLOHFHXAW23GJDTQIK2FRYUGS2IHAZHGPDUIBXDQ3K5HQWHISSUJA3TGNLOGMWFMKRJHAZHIS2WIYSUCU3NHYQT4SS4I47CSUR4IFICE4ZPG5VFMTLJIE2FIIKVHEXDGOZ7IVESSK3DGNEUMLZZG5YT4WJDI46VUMKVHUVEALCDHBJTYMZ2IBYCWK3TG5YTMM2YHVAFU4JSJFLWKNCKIVBEUOBTIE2C6RTUG5VHESJXFY3WGRK7IE4CESSFHNETISBYIEZEQO2WHAZHE5BEIFHWKLLPHA4S4M2JIREUSLKJIFICEM3RHA3UOW3PIUUEI4ZCGNDTU32OIBITQP3UGY7S2QCRG5YUOUS3FQ6WELK6H47UUZJJHVBEAKCFGZYVINC2G5VGARS6IE2EYM2IGMTTAWKTGU6TQ4CPFNNTQRZ3G5VFOSRNIUUEIMZWHRDSWZSKJA7XGRRBIBKFUOKCG5WV6MSYIRDGI4K2HNFXIXCZGZXUWJ2VFUSGU3CHG5YTK3SSIUUEIMZNHFIUKWZUGBHCSRSPIFGCOU3KG5YUQRTBFRMTEKRHHVCDUODOHBJD6TZRGMTD232NG5VF6OBOIFHWOPBQGQQVENTAGNBE2JKKGMVCK4J7HA3UQNCEI46VCMJDGNDFARJIJA3TE3DDFYRCGIJZG5YUON2BFU5F4QSTGNDGEVDIG5JSGJJCFUSGU3CHG5YUQK2VIQVUQWBRHFIUKV2PIY7EAYZKHYQUQLBUG5YT4WDMI46U65CDGFGUWP2HIVBEULBPGZZEUQC4G5VFMTLJHVOCCJJ5GIXVS33RFRMWSWKVFU4FIISAG5YT2R2EIYSTSYSBHBJSQRBHHBHFGMLGFVYDCJKGHAZHE5BEIQVUUKDIIA6HIRLMIY7HIZCAIFISC5JCG5VHEYKYIUUD2SSUGEVG6RR6GJDVGRLEIA3WEMSXG5YUOUZJIVPSOQKPGZLXK5JMHFTD64S5IA2SQZLRG5VFIWRSIUUD2SSTJFLUA5BGFVYTO3CAGMUSEXR3G5WW6P3FFRMSQMJPGFGUEOLJHBGGKUSKFNQTK4LIHAYF2ZS6IZAFYRCHIIWUEZRLGBIHIXDUIFICE5BIHAZHEQ3JIRQXKLKCGBITAS3KGBHV2NLTGY5VM3BNG5YUQKZYFR2EGP2XFU7SSSJFIRDVAIR4GMUCG32BHAZSMQJUIZAFI3RWFZLF6SJ7H5XUIWDHIBLTKTBDHA3T4KJPIIYUMN2UH5LWM3JQHRQE6QRXIBZFQIZFG5VFIWRZHRPSIWJMJB2UIUDCHJFFA3RCIFJFKJJOG5WV4VCRIJGGYVBXHIVTQXBEGBEWQJZUIEZEO5KOG5YUON2SFRMSSQSHHFIDM2ZKIVAD22SZFNMGM4BLHA3FERJVIYSTSYSBHRDWENRZHVQEUNCJFUQXEY2RG5VF44LEIUUD2RDBGVAESJTOIY7FYLBQGZ2C433PG5VGARS3IZAFEYJEGEVT4YKSGVAGKQDSIFJFWTR2HAZHE2CFIFVTKM2HG42CQTLNHNPWSOC5GZJXCU2NG5YT4WJDI52DS3ZUHFID642WGZKSYS22FU7WSQBTG5YT4PLGI52DERR3GQRDGWTGGBHUCV3FFU4FIISKG5VF6ZKAIMXEGWZEGEZUQKLVIA6UKL3EGV2DIZZ3G5YTMTTBINSSK5K5JBZTAJ2MHNFEYIZ4IFJFWRCEHAZW6LRMIVBVQTDIGVAXGJK3GBEUCLB2IBYVWLDTG5YT4PLJI46U6JTSJBNE2XBIGZXGSUSNFU5EEWJSG5WV2XTDHVNXGY3BFZZGKKTIHJEVIRTUHYQUOZRIG5WW6JCVIMXE2YTFIEYEMSCTHBHFGN3JIA4F6LTLG5YUQK2VFVYT4VKQGFGGA3LAHBKDK4KGIFKFIZKKG5YUON2BFQ6WYIKMGBIDGZ3DGFSSSK2LGVZE2U24G5YUQRTBIRRCCLDBINGWYWTGHBIV4KBJGZXGM2JPG5VF4VS3IBJGU4ZHHFIC2Z25HJFU2YJSGMTDYJ3QHA3UOW3VFY3WGRKIHJHGMNR6HNETIQRWFU6W65DVG5VF6OBLIA3VQ2SRIRQS2Y2QHRPFKUTLIFICE4ZUG5WXANZFIRRCCLJWGRCSCXZ2IRDTIUZSFU5EET3TG5YUQRTBIVPSKRJPHBIW4VSPGVBSESSEFU5T6ORXHAYE4MJVHVOCCIRHHRDVSMZXHBJXEMZVIBMDEUJZG5VHASTHF5ICK2LDIA5VGTKHINECU2BXFRYUGQSQHAZHIS2WIJUDCM2XJBZGS2SCGVAEULTPIFICIMSTHAZHGOSNIBJGSZCNI5YD42SHGJDTKUDLGMUCG3ZWG5YUOUS3FRMTEPZIII2GUXS7GRAE6MC2FNNTQRZMG5WV4VB6IA3U2VTJFVZTAI2EHRBGU5DAGZXGM4RVG5WW64ZJI4RDMKBCGFGGS4BEJA3TE4TFIFKFI3SVHAZHE5BEIRDFWKKLFU3SSLZPGZXGSVKOIFISC5JBG5WW6QBEIQVUULBKGY3SYNRMIFHCEMDMGMVVSIK2G5YUON3VIZOCG5BSHIZS632QGFGEOKLLGY6TWVC6HAZW6LRGI4RDMKZYGQVDIITTFVYW4LCBGV2DIS3UG5WWMKZDIBXDQ3LGHUVC24KZJA7XGJ3LGZZEUQC5HAZXAP22JA5E2SKQJB2EO3C2HRQDIJZRIFJVQL2EHAZHIZTII4RDMKTIG42DUWKRIFICKJRZFNQTKX2GG5YUOU2JIRRCWPRVHAYUSLTJGU7U2X3MFV2SWKSJHAZSMZDJI4RDMKZYH47GSQDQGBICEXLCIBKFUS2IHA3UQ2S4GRAF4ZRYGEVUOZJKHBHFMI3CIA2SSLBMG5YUQZBPIRRCS2KQHVBWWJBUHVAV4UB4IFJFKJJOG5WWMKZDIJUCQUK7H43HEKRQIBIHEJ3PIBZFO3ZOG5YT4WDPIFHWMJ3JFROT6NC7JAREUULTFVYDCJKGHA3FKNBPI4RDMK2BH47UCXBKHBKUI2SWGMUCG32AHA3UQT2QFQ6WGNSXI47UAQCIIA6UKNLFGY6TYL3SHA3UOQDDI46VULR7I4SWCRDJGU7FA326IFICE4ZPG5WVYZC5IE2FE2ZOG5LUG4CIFRLTATZSGZXGM4RUG5YUOU2JIZAFGWLPIZCFQUKXHBJTW2RRGMUCG3Z7G5VF44LUIFHWOPCSGIXWYKKHHFHUIZBRHYQUOZRHG5VF6ZKDIBXDQVRNGY7TKUZNHNGFOUSUGY3WU3JIG5YT4WDPI4RD4ZJ2GVAESJTNGZYTC4S6IFJFWTRYG5VF6OBIIYSUCODBJB2U2UZGIFICKIZXIBYV243FG5YUON2SFY3VUY3VGEZGMWSMIJEWILSUGMTDYWZDG5VGARTHI52DAMBMFVZUELZEGZKGKSKBIA3CKYSAG5WVYZDGIE2EYMBSHRDSWZZUHRPVELLSFRYUGQTAG5YTK3SSIUUDYO2NFU3SSLZPGZXGSN2EIFICIM2KHA3UQNBYGRAGA42TGQUTOQKIIFHV4XJRIBYGQMZYG5VFOSJTIYSUCODCGBIUWXLOIIZGOPCNFVYDA4KBG5WVYQCUIIYUQTRJGQVDIIZFGBHGARLFFU6W6YTKHA3FKM3UIFVTILRMIE3W4RCEIA6UKKLBFU7GY4JPHAZSSJSXFVYUQWBCHYTVOVLCIFICIRJGFRYUGQSQG5WWMS3GIZAFI3TBGMSHCMJCFVZDI5CVFUQXEMBKG5VFOSRNINSSG3LIGIXWYJSQJA6SQWK6GY3WWUJ3G5VGARTKJA5E2T2FJFLTO2ZJI47UCVCOGMUCKJKTHA3UQTZ6I5MHGZK5JBZHE3KNHBJD6SJPGMVVSWTPG5YTM2LRIFHWKLLPG46F6OKHI45SULDIIE3CSQRXHA3UQT2KIYSTSYRYG5LV6LLFH5XSSRTDFY2WWQCUG5YUOUZJFU5F6UK2FYQSK4CGGJDFG4S7FY2WUZ2KHAYE4MJYIA3U63J3JFLVGKDLFRMFCXKGGZACWI26G5YUQRTBIQVT64BLFV2D4YSZIA4XINSNGZYUWUTBHA3FERJYIA3VOQDIGZLXK5KIIVOEY33MIFICE4ZPG5WVYZC5IE2FKMBBGY3SGLDJHNPWSNK4GY3VWRTMG5YT4PLJI46VSISYGY3VSUJ3HNCWGQTTFU4FKJCYHAYEWQR6IE2EYMBSHFHWOUSBGJDVGNS7GNEUKZJUHA3T4KJPIIYUMQDMGIUDWJJMHNCUQOLUGMVCMTC2HAZXAP25IVPHGWKAHFID64C7F5GWQJBQIFIHIVBXG5YTM2SQIFHW4ZCKGVBCOKZ5JA3TE2LCGY7S4QS6G5YUON2SIVPHGXC7HE3EKZBPFVYVGSSNGZACUIKNG5VFMTJJHUST6ZJFHNFCMRLFIBJD4OJQFQ5TWZRKG5YTMNCWHUST2VCUHJGTYNZQHBHFK4TAGNBE2XBOG5VHERZNFRMSOOBIJFHS4TSFGVAEU2ZOIFJVQOCNG5VSIPTOFZJSEJTDGEZUQKJYGRAGUXLFIFJVQJKJG5YUON3VFR2EEQ2CHURFC4BPHJUXAUCFGZKSYYJ6HAZHIS2TIIYUQTRJGQRDGWTGGBHUCXLIIBMDEWZQG5YTK4DCIZNW4ZKVGNEDOUBYJA3TE4TGGMUSSKKJG5WVYZDAIA3U6ZZYGJFU2QRBHRQGUJZLIFIHK2KWG5WV6MS3HVAFQYTCGITWWZJFGBHVYV3DIA3WEPCMG5YUQSDOIZAFGX2RFQ4V2ULFHVQEUNCJFU7GYVLNHAZXAP22I52DERR3GQRDGWZRGZXUUXSLGZKSY2R2G5VHELBBFR2EGP2XGQVCC2JGHBJD6SJPGV2DIS3SHAZSSJSXFVYUSTSCGBICCWZFFRLUWZZXIFISC5DVHAYE4MJ3IBJHCWLEGFHCYYZTIM2DUK3NGMUCMKBJG5YUOUS3FR2EGORYGVBDAMK3JA6SQU24FVYDCLKQG5WWMS3GIYSTOTZGFVZTAI2MG5JT42RXHYQUOZRIHA3FIUTCIVPHGXCWGMWVE3BHHFUF2NDKIFKFIZKOHA3T4KJGINSS45JLIUVXCZTIIRDWWWCKIFICE5BKHAZW6LRDI4RD6KCFHIZV2OJ5H5XGGN3BFYQS4RRBG5YUONZ5FR2EGQBUG42GQIZ7HFUCOMLPFV2SWKKUHA3UOXBGI4RDG2RJFVZUEK3GIFICISBHGY5VMYZ6G5VSIPTIFY3WI33OINQTER25HBGGKSKGIBZFO3ZXHA3UQ2SWFZXDYIJ2HA4USSJMIRDG4SRSIFICIM2OHA3UOJK5I4RD4ZK4GZNFSZJFIFGUCKTPGY7SYLBMG5YUOUS3FRMTEPZIII2GUXS7JBYTKWKWHYTEQXKJG5WV43B3HVAFU3RGHJHSGQLUGBFXGWKNGZYUYXJRG5VGARS6I4RDMKB3HBJV42BMHRQE44ZMGV2TCUROG5WXAZSEFU5GQO3HHJHGMNLQJA6SQL2QIFJVQJKOG5YTIJRWIBJGQXTQFZXW6L2PHNGDYPKRGYQS4X24G5YUONZ5FR2EGQBUG42GQISVJA7XGJDLIBMDEWZPG5VSIPTLIUUD2SRSGQVDIISSJBYTQRKPGMUSEXR6G5VF4VTBIJGGWKR5H5MGGTR2HNGFOT2UHYSUWXRKHAZXAJCLI52DWQZ7HMVDOQSZJARGQQ3LIE3CSQR2HAZHGPDUIZAFEWSHGBITASBQIYVSYRJKFUSGUY2RG5YT4PLJINSS4YCAGFGHGILEGZXWMNSXGZ2C6VBLHA3T4KJDIZAFI3SOH47TQVSIHVQESUBWIBZFOYZJG5YTMTZEIZAFYRJ6HA4HCKBRHBHFGQDLGZXGOZKUHAZHE2BZIZNW2Y3SHFICIYLVIFHD2M3JIFICE5BIG5VF6ZKGIZNW4ZKVGZNGE2ZGIRDFGLZMFU6XAKDMG5YUQSJGIUUDYO3GJFHVEZRSI45UMLZNGZXGM4RWHA3FIISYHRPSIXZ2GIXVS4SFHFSV4TSXIBMDEUKLG5WV4JDMIMXEEVJMJB2WQZLHHFUF2KDGIE3CSXKDG5VG6V2CIRDGKNJUHQWSQU3SHJGGKSZ2IFICE4ZRHAZHIZTLIZOCCYCLHVCEYSB2IIZS2WSVGNAT25KZG5YUQK2YIRDFUJDLHVCVWNDRJBXU4TSGGV2TCWZQG5YUK4RUIVBVQUBWHFID64JNINDUYJZHFU7WSSJTHA3T4KJPIIYU6NZEHNFC6TBSHBGD4YCSIE3CSSZZHA3FKNBJIJGGWKSWJBZGS2SCI46WGSJ4IE2SY2SGHAYEYMTLIFHXAIKCGNCVY3K6HJFTEQBKHYQUQLBXG5YUON2SIVPHCRRFI47HCKCCFZWUS4SCGZXGOKJ6G5YT4PLAIZNW65B2HFIC2ZZ5I45UKQLMGY5FS4BSG5WV6TLEIBJHG4ZIHEXEKR2DHNETCXB7IBYVWOJSHA3UOW3SFVYT4VLRHBKG2V2MGBGWGZC4IFIHIVJSG5YTKW2JHYREIZ2GHBJUYXSUGFESGZKKGZKSUJR6G5YUQLLFIRQXKLKLG5LXCOLJHNGFKI3BGY3WWUJ3G5YTMM24IA3U63JMGQQVENTAGNBGKY24IBLDQWDKHA3FI3RGIIYUQSZMGZMUU5JEIY7FYIZOGVZE2L2TG5YUQK2YIQVUCLBJGNDHIXLKJA3TGXJFIFJFKJJMG5YUON2SFY3WGSKIGMSGQJ2DGRAGUV3EGVZE2JSPG5YUQSJJIZOCCWR7HJHF2MB4GZXWMP22GY3VWRTQG5YTM2LOINSSMJRXGQQVENS7GVBFYMSAIBLDQYS6G5YT4WDMIZAF22B4I4SWUSCYHBJD6SJPGV2DIXTOHAZHEITRI4RD4ZKKGFGGS4BHFRMTE4SHIFJFG3TDHAYF4LDQINEWQ32JHQSTURSCI46UQOR2FU6DYXKKG5VFOSJTIVPHESBIGZMHEVRXFRLS23Z5GZXGM2JPHAYEWQSRIBXDCKRFGQQW2S2YFVZGU2CMGMSS4WZ7G5VHELBEFY3VSZ2GHE3GA4Z3HRQE6PZXIBLTMJCFHA3UOJKUI4RDG4TRJFLTO2ZIJA3U4JTFIFICE5BIG5YUQZBYIZOCGXC3GZND4UZCFVZDKISWGY5VM5J4G5YUQSJJIZOCG4JUIUWCK4CQHBJTYMZ3HYQUQLBJG5VF4VTBIJUCOSLRFVZTAIZCFVZTCMKTHYQUOZRHG5VGARTKIMXEEUSXH5NECU2JHBHFGN3JGMTVMJLGG5WWMTDEFRMSOLZMFVZHE3CKHVASUZZNIFJVQVSQG5VF6ZJ2IZNW42DUJFLWKN3LHVAUK2RMIBLDQ2ZNG5YUOUS3FQ6WCKCUGJFV6UCNHROG4WLBFYQS4M3NG5VF4JSIIA3U63KFGIXWYJSQFNNCCZJFFU6DYS2KG5VG6V2MIZNW65BBGBIDY3LEHFUEETJBGMSS4URUHA3T4KJGIZOCG4DQHVBXIKLAGFEFY2JSIFICE4ZPG5YUONZ5FNODIUSK'
a = [t]

for _ in range(9):
    for decoder in decoders:
        try:
            t = decoder(a[-1])
            assert t.isascii()
            a.append(t)
            break
        except:
            pass
# print(a[-1])
print(b64decode(a[-1]).decode())
~~~