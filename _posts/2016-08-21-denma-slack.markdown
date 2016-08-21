---
layout: post
title: 덴경대 집합! for Slack
---

# 덴경대 집합! for Slack

![](http://cfile28.uf.tistory.com/image/27690D3F55FD912D042E8F)  
_덴경대 집합!_

인기리에 ~~자유연재~~를 하고 있는 네이버 웹툰 [덴마](http://comic.naver.com/webtoon/list.nhn?titleId=119874)가 나오면 누구보다 빠르게 집합을 하고자 만들게 되었다. 

이미 오래전부터 덴경대 앱들은 앱스토어에 많았지만, 어느순간부터 동작하지 않기도 했고, 슬랙봇으로 만들어 신작이 나오면 다 같이 보고 의견을 나누었으면 좋겠다는 생각이 들어서 슬랙봇으로 만들게 되었다.


# 1. 파싱

덴마 최신화가 올라왔는지 확인하려면 덴마 웹툰 리스트를 가져와서 마지막에 나온 것이 최신화인지 확인하는 처리가 필요하다.  
네이버에서는 웹툰이 나왔는지 확인할 수 있는 API를 제공하지 않기 때문에 특정 주기마다 최신화가 나왔는지에 대한 처리가 필요하다.

덴마의 URL은 `http://m.comic.naver.com/webtoon/list.nhn?titleId=119874`이다.  
(PC 페이지는 광고가 달려있기 때문에 깔끔한 모바일 페이지를 이용했다.)

해당 페이지에서 요소보기를 하면 아래와 같은 구조로 되어 있는 것을 확인할 수 있다.

```html
<ul class="toon_lst lst2" id="pageList">
  <li>
    <div class="lst">
      <a onclick="nclk(this, 'lst.episode','119874','1');" href="/webtoon/detail.nhn?titleId=119874&amp;no=895&amp;week=sun&amp;listPage=1">
        <span class="im_br"><span class="im_inbr">
        <img src="http://thumb.comic.naver.net/webtoon/119874/895/inst_thumbnail_20160819173515.jpg" width="71" height="42" alt="">
        </span></span>
        <div class="toon_info"><h4>
          <span class="toon_name"><strong><span>
          2-570화 3.The knight(98) 
          </span></strong></span>
          <span class="aside_info"><span class="ico_up">UP</span></span></h4>
          <p class="sub_info">
            <span class="st1"><span class="u_grp"><span class="u_grp_v" style="width:89.89%"></span></span></span>
            <span class="if1 st_r">8.99</span><span class="bar">|</span><span class="if1">16.08.20</span>
          </p>
        </div>
      </a>
    </div> 
  </li>

  ....
</ul>
```

ul 태그의 toon_lst라는 클래스 아래에 li 요소들로 각 화들이 묶여있고, li 아래의 a 태그에 각 화의 url이 지정되어있고, span.toon_name 아래에 각 화의 제목이 속해있다.  
그리고 맨 위에 있는 요소가 최신화이므로 맨 첫번째 요소만 파싱해서 저장하기로 정했다.

필자는 최신화인지 확인하는 요소로 최신화의 제목을 저장하여 비교하는 방법을 사용했으므로 최신화의 제목과 url을 가져오는 코드를 작성했다.(최신화의 ID 값을 저장해도 된다.)

```ruby
def get_last_title
  get_denma_data.css("ul.toon_lst > li")[0].css('span.toon_name > strong > span').text
end

def get_last_url
  'http://comic.naver.com' + get_denma_data.css("ul.toon_lst > li")[0].css('a').attribute('href').to_s
end

def get_denma_data
  Nokogiri::HTML(HTTParty.get('http://m.comic.naver.com/webtoon/list.nhn?titleId=119874').body)
end
```

최신화의 제목과 url을 가져오고 나서 해당 화의 제목을 추후에 최신화인지 확인하기 위해서 파일로 저장해둔다.

```ruby
def set_log_title(title)
  f = File.open('./denma.txt', 'w:utf-8')
    f.write(title)
  f.close
end
```

최신화인지 확인할때 기존에 파일에 저장된 최신화의 제목을 가져오는 코드도 작성한다.

```ruby
def get_log_title
  begin
    f = File.open('./denma.txt', 'r:utf-8')
    f.readline
  rescue 
    return ''
  end
end
```

(로그파일에 저장된 것이 없으면 빈 문자열을 리턴한다.)


# 2. 비교

덴마의 최신화 정보를 파싱해와서 저장하는 작업까지 끝났다.  
이제 남은건 가져온 덴마 최신화의 정보가 새로 나온 것인지 확인하는 부분이 필요하다.

새로 나온 덴마인지 확인하기 위해서 위에서 작업한 요소들을 활용한다.

```ruby
def run
  title = get_last_title.gsub(/\s+/, "")
  url = get_last_url
  log_title = get_log_title.gsub(/\s+/, "")

  if title != log_title
    set_log_title(title)
    result = send_to_slack(url)
  end
end
```

현재 로그 파일에 저장된 제목을 가져오고, 덴마 URL에서 최신화로 등록되어있는 제목을 가져와서 비교를 한 후, 두 제목이 다르면 최신화가 나왔다는 이야기이므로 슬랙으로 정보를 보낸다.


# 3. 주기적으로 확인하기

모든 동작들이 완성되었으니 이젠 주기적으로 최신화 덴마가 나왔는지 확인을 해야한다.  
필자는 5분마다 최신화 덴마가 나왔는지 확인하기 위해서 아래와 같이 cron을 등록하였다.

```
*/5 *   *   *   *   /usr/bin/ruby /YOUR/DENMA_REPOSITORY/DIRECTORY/denma.rb
```

혹시나 rvm을 사용하고 있다면 아래의 명령어를 한번 실행시켜줘야 한다.

```
rvm cron setup
```

이제 덴마봇은 매 5분마다 최신화가 나왔는지 확인을 하고, 최신화가 나왔으면 로그 파일에 최신화 제목을 저장한 후 SLACK으로 최신화 정보를 전달한다.


# 4. 덴경대 집합!

모든 준비가 다 되었으니 이젠 덴경대를 호출하는 일만 남았다.

슬랙으로 메세지를 보내는 방법은 여러가지가 있지만 필자는 Incomming Webhook을 사용하였다. [참고](https://api.slack.com/incoming-webhooks)

```ruby
def send_to_slack(url)
  data = generate_message(url)
  payload = { 'payload' => data.to_json }

  HTTParty.post(@@SLACK_URL, body: payload)
end

def generate_message(url)
  data = {
    'username' => '덴경대',
    'text' => "<#{url}|덴경대집합!>"
  }

  return data
end
```

Slack에서 Incomming Webhook을 생성하면 Webhook URL이라는 것이 생기는데 이 URL을 @@SLACK_URL의 값에 넣어주면 된다.

**잘 활용하고 있는 예시**  
![](../../../img/call.png)

# 소스코드 및 문서

[https://github.com/Akkiros/denma](https://github.com/Akkiros/denma)
