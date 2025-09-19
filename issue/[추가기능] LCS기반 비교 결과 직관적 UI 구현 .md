```
3가지형태의 문자열비교
1. JSON
2. HTML
3. String (deletemeter 포함)

비교전략
줄단위 비교로 해당 1,2,3 형태 전처리

```

1. java-diff-utils의 patch객체활용
2. patch객체의 delta속성의 4가지 diff 형태 활용
3. 2번을 통한 beyound compare 줄단위 비교 UI 구현
    - 줄 맞춤
    - 동일한 문자열이 아닌 String 앞에 구분자 ; 추가

###### 구현예시

```java
private Map<String, Object> makeDiffMap(String origin_res,String res) {	
		try {
			Map<String, Object> result = new HashMap<>();
			if(origin_res.isBlank() && res.isBlank()) return Map.of("origin_res","","res","","result",true);
			
			// json 여부 체크
			boolean isJson = origin_res.trim().startsWith("{") && res.trim().startsWith("{");
			// html 하나라도
			boolean isHtml = origin_res.trim().startsWith("<") || res.trim().startsWith("<");
			
			/* 줄단위 비교를 위한 전처리 */
			// json 전처리
			if(isJson) {
				origin_res = origin_res.replaceAll("\\R","").replace(",",",\n").replace(":[", ":\n[");  
				res = res.replaceAll("\\R","").replace(",",",\n").replace(":{", ":\n{").replace(":[", ":\n["); 
			// html 전처리
			} else if(isHtml) {
				origin_res = origin_res.replaceAll(">(\\s*)<", ">\n<");
				res = res.replaceAll(">(\\s*)<", ">\n<");
				origin_res = origin_res.replaceAll("\\R", "\n");
			    res = res.replaceAll("\\R", "\n");
			// delimeter | 전처리 
			} else {
				origin_res = origin_res.replaceAll("\\R", "\n");
			    res = res.replaceAll("\\R", "\n");
			    origin_res = origin_res.replaceAll("|", "\n");
			    res = res.replaceAll("|", "\n");
			}
			
			List<String> linesOrigin = Arrays.asList(origin_res.split("\n"));
		    List<String> lines = Arrays.asList(res.split("\n"));
		    
		    // html 비교제외처리
		    if(isHtml) {
		    	MessageHttpClient mhc = new MessageHttpClient();
		    	Set<Integer> originExcludes = mhc.findExcludeIndexes(linesOrigin);
		 	    Set<Integer> excludes = mhc.findExcludeIndexes(lines);
		 	    linesOrigin =IntStream.range(0, linesOrigin.size())
			            .filter(i -> !originExcludes.contains(i))
			            .mapToObj(linesOrigin::get)
			            .collect(Collectors.toList());
		 	    lines = IntStream.range(0, lines.size())
			            .filter(i -> !excludes.contains(i))
			            .mapToObj(lines::get)
			            .collect(Collectors.toList());
		    }
		    		    
		    // 비교 
		    // controller의 반환 result.data에 해당함수 result를 value로 넣을거임
		    Patch<String> patch = DiffUtils.diff(linesOrigin, lines);
		 
		    // 1. patch객체의 delta로 diff형태를 파악하는데 순서보장을 해야함
		    List<AbstractDelta<String>> deltas = new ArrayList<>(patch.getDeltas());
		    deltas.sort(Comparator.comparingInt(d -> d.getSource().getPosition()));
		    
		    // 2. linesOrigin가 기준인데 비교할때 linesOrigin, lines 둘다 줄을 맞춰줘야함 beyoundcompare처럼 		    
		    //  정렬된 patch객체를 통해 다를때,없을때,추가내용일때 Stirng맨앞에; 붙이고 같을때는 그냥 넣도록  List의 요소를 수정해서
		    // 비교한 위의 내용을 반영한 list두개가 나와야함

		    // 정렬/정렬결과를 반영하여 양쪽 라인을 맞춰서 생성
	        List<String> alignedOrigin = new ArrayList<>();
	        List<String> alignedRes    = new ArrayList<>();

	        int oIdx = 0; // originRes 포인터
	        int rIdx = 0; // res 포인터
	        for (AbstractDelta<String> d : deltas) {
	        	Chunk<String> src = d.getSource();
	        	Chunk<String> rev = d.getTarget();
	        	 int srcPos = src.getPosition();
	             int revPos = rev.getPosition();
	        	
	        	 // 1) delta 이전의 동일 구간 채우기 (변경 없음)
	            while (oIdx < srcPos && rIdx < revPos) {
	                alignedOrigin.add(linesOrigin.get(oIdx));
	                alignedRes.add(lines.get(rIdx));
	                oIdx++;
	                rIdx++;
	            }
	            // 포인터가 어긋난 경우 남은 쪽을 빈 줄로 보정
	            while (oIdx < srcPos) {
	                alignedOrigin.add(linesOrigin.get(oIdx));
	                alignedRes.add(""); // 상대 라인 없음
	                oIdx++;
	            }
	            while (rIdx < revPos) {
	                alignedOrigin.add("");
	                alignedRes.add(lines.get(rIdx));
	                rIdx++;
	            }
	        	
	            // 2) delta 구간 처리
	            List<String> srcLines = src.getLines();
	            List<String> revLines = rev.getLines();

	            int max = Math.max(srcLines.size(), revLines.size());

	            switch (d.getType()) {
	                case CHANGE:
	                    for (int k = 0; k < max; k++) {
	                        String left  = (k < srcLines.size()) ? srcLines.get(k) : "";
	                        String right = (k < revLines.size()) ? revLines.get(k) : "";
	                        alignedOrigin.add(";" + left);  // 변경 라인 표시
	                        alignedRes.add(";" + right);    // 변경 라인 표시
	                    }
	                    oIdx = srcPos + srcLines.size();
	                    rIdx = revPos + revLines.size();
	                    break;

	                case DELETE:
	                    for (int k = 0; k < srcLines.size(); k++) {
	                        alignedOrigin.add(";" + srcLines.get(k)); // 삭제됨
	                        alignedRes.add("");                       // 상대 없음
	                    }
	                    oIdx = srcPos + srcLines.size();
	                    // rIdx는 그대로(삽입 없음)
	                    break;

	                case INSERT:
	                    for (int k = 0; k < revLines.size(); k++) {
	                        alignedOrigin.add("");                    // 상대 없음
	                        alignedRes.add(";" + revLines.get(k));    // 추가됨
	                    }
	                    // oIdx 그대로(삭제 없음)
	                    rIdx = revPos + revLines.size();
	                    break;

	                default:
	                    // 안전장치: 예상치 못한 타입은 변경으로 처리
	                    for (int k = 0; k < max; k++) {
	                        String left  = (k < srcLines.size()) ? srcLines.get(k) : "";
	                        String right = (k < revLines.size()) ? revLines.get(k) : "";
	                        alignedOrigin.add(";" + left);
	                        alignedRes.add(";" + right);
	                    }
	                    oIdx = srcPos + srcLines.size();
	                    rIdx = revPos + revLines.size();
	                    break;
	            }
	        }
		    
	        // 3) 마지막 남은 동일 구간 push
	        while (oIdx < linesOrigin.size() && rIdx < lines.size()) {
	            alignedOrigin.add(linesOrigin.get(oIdx));
	            alignedRes.add(lines.get(rIdx));
	            oIdx++;
	            rIdx++;
	        }
	        while (oIdx < linesOrigin.size()) {
	            alignedOrigin.add(linesOrigin.get(oIdx));
	            alignedRes.add("");
	            oIdx++;
	        }
	        while (rIdx < lines.size()) {
	            alignedOrigin.add("");
	            alignedRes.add(lines.get(rIdx));
	            rIdx++;
	        }
	        
	        result.put("originRes", alignedOrigin);
	        result.put("res", alignedRes);
		    
			return result;
		} catch (Exception e) {
			logger.error("makeDiffMap - DIFF ERROR : ",e);
			return Map.of("result",false,"message",e.getMessage());
		}
	}
	

```
