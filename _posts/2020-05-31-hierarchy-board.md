---
layout: post
title : "Spring Boot + Mybatis + Mysql 을 활용한 계층형(=hierarchy) 게시판 구현"
subtitle : "2020-05-31-hierarchy-board.md"
date: 2020-05-31 14:00:00
author: "전봉근"
header-img: "img/posts/android-bg.jpg"
comments: true
tags: [java, springboot, mybaits, mysql]
---

## Spring Boot + Mybatis + Mysql 을 활용한 계층형(=hierarchy) 게시판 구현
----------------------------------------------------------------

- 테이블 설계
  ```
    CREATE TABLE `tb_board` (
        `BOARD_NO` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '게시판 고유 값',
        `GROUP_NO` int(11) unsigned NOT NULL COMMENT '게시판 그룹번호 (원글은 자신의 값)',
        `SORT_SEQ` int(2) unsigned NOT NULL COMMENT '게시글 정렬 순번',
        `BOARD_LVL` int(2) unsigned NOT NULL COMMENT '게시글 레벨(depth)',
        `BOARD_TITLE` varchar(45) NOT NULL COMMENT '게시글 제목',
        `BOARD_CONTENTS` text COMMENT '게시글 내용',
        `SYS_REGR_ID` varchar(20) NOT NULL COMMENT '시스템 등록일',
        `SYS_REG_DTIME` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '시스템 등록일자',
        `SYS_MODR_ID` varchar(20) NOT NULL COMMENT '시스템 수정자',
        `SYS_MOD_DTIME` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '시스템 수정일자',
    PRIMARY KEY (`BOARD_NO`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='일반 게시판';
  ```
  
- Entity Class 작성
  [Board.java]
  ```
    package com.example.bkjeon.feature.board;

    import com.fasterxml.jackson.annotation.JsonFormat;
    import lombok.*;

    import java.time.LocalDateTime;

    @ToString
    @Builder
    @Getter
    @NoArgsConstructor
    @AllArgsConstructor
    public class Board {

        private Long boardNo;
        private Long groupNo;
        private Integer sortSeq;
        private Integer boardLvl;
        private String boardTitle;
        private String boardContents;

        private String sysRegrId;

        @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
        private LocalDateTime sysRegDtime;

        private String sysModrId;

        @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
        private LocalDateTime sysModDtime;

    }
  ```
- DTO Class 작성
  [BoardDTO.java]
  ```
    package com.example.bkjeon.feature.board;

    import lombok.AllArgsConstructor;
    import lombok.Getter;
    import lombok.NoArgsConstructor;

    import javax.validation.constraints.NotEmpty;

    @Getter
    @NoArgsConstructor
    @AllArgsConstructor
    public class BoardDTO {

        @NotEmpty(message = "제목을 입력하여 주시길 바랍니다.")
        private String boardTitle;
        private String boardContents;

        @NotEmpty(message = "userId 값을 입력하여 주시길 바랍니다.")
        private String userId;

    }
  ```
- Controller 작성
  [BoardController.java]
  ```
    package com.example.bkjeon.base.services.api.v1.board;

    import com.example.bkjeon.feature.board.BoardDTO;
    import io.swagger.annotations.ApiOperation;
    import io.swagger.annotations.ApiParam;
    import lombok.AllArgsConstructor;
    import org.springframework.validation.BindingResult;
    import org.springframework.web.bind.annotation.*;

    import javax.validation.Valid;

    @RestController
    @AllArgsConstructor
    @RequestMapping("boards")
    public class BoardController {

        private final BoardService boardService;

        @ApiOperation("게시글 리스트 조회")
        @GetMapping
        public List<Board> getBoardList(
            @ApiParam(
                value = "page 번호를 설정할 수 있으며 설정 값은 1-N까지 입니다.",
                name = "page",
                defaultValue = "1",
                required = true
            ) @RequestParam int page,
            @ApiParam(
                value = "페이지 별 레코드 갯수를 설정 할 수 있습니다.",
                name = "size",
                defaultValue = "10",
                required = true
            ) @RequestParam int size
        ) {
            return boardService.getBoardList(page, size);
        }

        @ApiOperation("게시글 상세 조회")
        @GetMapping("{boardNo}")
        public Board getBoard(
            @ApiParam(value = "boardNo", name = "boardNo", required = true) @PathVariable Long boardNo
        ) {
            return boardService.getBoard(boardNo);
        }

        @ApiOperation("메인 게시글 등록")
        @PostMapping
        public void setBoard(
            @RequestBody @Valid BoardDTO boardDTO,
            BindingResult bindingResult
        ) {
            boardService.setBoard(boardDTO, bindingResult);
        }

        @ApiOperation("서브 게시글 등록")
        @PostMapping("{boardNo}/replies")
        public void setBoardReply(
            @ApiParam(value = "boardNo", name = "boardNo", required = true) @PathVariable Long boardNo,
            @RequestBody @Valid BoardDTO boardDTO,
            BindingResult bindingResult
        ) {
            boardService.setBoardReply(boardNo, boardDTO, bindingResult);
        }

    }
  ```
- Service 작성
  [BoardService.java]
  ```
    package com.example.bkjeon.base.services.api.v1.board;

    import com.example.bkjeon.base.common.enums.ResponseResult;
    import com.example.bkjeon.base.common.model.ApiResponseMessage;
    import com.example.bkjeon.feature.board.Board;
    import com.example.bkjeon.feature.board.BoardDTO;
    import com.example.bkjeon.feature.board.BoardMapper;
    import lombok.AllArgsConstructor;
    import lombok.extern.slf4j.Slf4j;
    import org.springframework.stereotype.Service;
    import org.springframework.transaction.annotation.Transactional;
    import org.springframework.validation.BindingResult;

    import java.time.LocalDateTime;
    import java.util.List;

    @Slf4j
    @Service
    @AllArgsConstructor
    public class BoardService {

        private final BoardMapper boardMapper;

        // 게시글 리스트 조회
        @Transactional(readOnly = true)
        public List<Board> getBoardList(int page, int size) {
            List<Board> boardList = new ArrayList<>();

            try {
                Integer offset = (page - 1) * size;
                boardList = boardMapper.selectBoardList(size, offset);
            } catch (Exception e) {
                if (log.isErrorEnabled()) {
                    log.error("getBoardList ERROR {}", e.getMessage());
                }
            }

            return boardList;
        }

        // 게시글 상세 조회
        @Transactional(readOnly = true)
        public Board getBoard(Long boardNo) {
            Board board = null;

            try {
                board = boardMapper.selectBoard(boardNo);
            } catch (Exception e) {
                if (log.isErrorEnabled()) {
                    log.error("getBoard ERROR {}", e.getMessage());
                }
            }

            return board;
        }

        // 메인 게시물 등록
        @Transactional
        public void setBoard(BoardDTO boardDTO, BindingResult bindingResult) {
            if (bindingResult.hasErrors()) {
                log.error("Valid ERROR {}", bindingResult.getFieldError().getDefaultMessage());
            }

            try {
                Board board = Board.builder()
                    .sortSeq(0)
                    .boardLvl(1)
                    .boardTitle(boardDTO.getBoardTitle())
                    .boardContents(boardDTO.getBoardContents())
                    .sysRegrId(boardDTO.getUserId())
                    .sysRegDtime(LocalDateTime.now())
                    .sysModrId(boardDTO.getUserId())
                    .sysModDtime(LocalDateTime.now())
                    .build();
                boardMapper.insertBoard(board);
            } catch (Exception e) {
                if (log.isErrorEnabled()) {
                    log.error("setBoard ERROR {}", e.getMessage());
                }
            }
        }

        // 서브 게시물 등록
        @Transactional
        public void setBoardReply(Long boardNo, BoardDTO boardDTO, BindingResult bindingResult) {
            if (bindingResult.hasErrors()) {
                log.error("Valid ERROR {}", bindingResult.getFieldError().getDefaultMessage());
            }

            try {
                Integer sortSeq = null;
                Integer boardLvl = null;

                // CASE 1: 원글의 GROUP_NO, SORT_SEQ, BOARD_LVL 기준으로 답글의 저장될 데이터를 계산한다.
                Board board = boardMapper.selectBoard(boardNo);
                if (board == null) {
                    log.error("잘못된 접근입니다.");
                }

                if (board.getBoardLvl() == 1 && board.getSortSeq() == 0) {
                    // 원글 그룹에 기존에 답글이 있는지 유무 체크
                    Long boardGroupCnt = boardMapper.selectBoardGroupCnt(board.getGroupNo());
                    if (boardGroupCnt == 1) {
                        // CASE 1: 원글의 답글 등록 (원글 그룹으로 조회했을시 개수가 1개일 경우)

                        /**
                        * 원글의 GROUP_NO, SORT_SEQ, BOARD_LVL 기준으로 답글의 저장될 데이터를 계산한다.
                        * GROUP_NO: 원글의 GROUP_NO 값, SORT_SEQ: 원글 (SORT_SEQ + 1), BOARD_LVL: 원글 (BOARD_LVL + 1)
                        */
                        sortSeq = board.getSortSeq() + 1;
                        boardLvl = board.getBoardLvl() + 1;
                    } else if (boardGroupCnt > 1) {
                        // CASE 2: 원글에 답글을 한개 더 작성하는 경우 (원글 그룹으로 조회했을시 개수가 2개 이상일 경우)

                        /**
                        * GROUP_NO가 같으면서 새로운 답글이 상위로 위치하기 위해서
                        * 기존 원글의 답글들의 SORT_SEQ를 전부 +1 한다.
                        */
                        boardMapper.updateBoardGroupSort(board.getGroupNo());

                        /**
                        *  GROUP_NO: 원글의 GROUP_NO 값, SORT_SEQ: 원글 (SORT_SEQ + 1), BOARD_LVL: 원글 (BOARD_LVL + 1)
                        */
                        sortSeq = board.getSortSeq() + 1;
                        boardLvl = board.getBoardLvl() + 1;
                    }
                } else if (board.getBoardLvl() > 1) {
                    // CASE 3: 답글에 답글을 작성하는 경우

                    /**
                    * GROUP_NO가 같으면서 새로운 답글이 상위로 위치하기 위해서
                    * 기존의 답글들의 SORT_SEQ를 전부 +1 한다.
                    */
                    boardMapper.updateBoardReplyGroupSort(board.getGroupNo(), board.getSortSeq());

                    /**
                    * GROUP_NO: 원글의 GROUP_NO 값, SORT_SEQ: 원글 (SORT_SEQ + 1), BOARD_LVL: 원글 (BOARD_LVL + 1)
                    */
                    sortSeq = board.getSortSeq() + 1;
                    boardLvl = board.getBoardLvl() + 1;
                }

                // 답글 저장
                Board insertBoardReply = Board.builder()
                    .groupNo(board.getGroupNo())
                    .sortSeq(sortSeq)
                    .boardLvl(boardLvl)
                    .boardTitle(boardDTO.getBoardTitle())
                    .boardContents(boardDTO.getBoardContents())
                    .sysRegrId(boardDTO.getUserId())
                    .sysRegDtime(LocalDateTime.now())
                    .sysModrId(boardDTO.getUserId())
                    .sysModDtime(LocalDateTime.now())
                    .build();
                boardMapper.insertBoardReply(insertBoardReply);
            } catch (Exception e) {
                if (log.isErrorEnabled()) {
                    log.error("setBoardReply ERROR {}", e.getMessage());
                }
            }
        }

    }
  ```
- Mapper.java 작성
  [BoardMapper.java]
  ```
    package com.example.bkjeon.feature.board;

    import java.util.List;

    public interface BoardMapper {

        void insertBoard(Board board);
        void insertBoardReply(Board board);
        List<Board> selectBoardList(Integer size, Integer offset);
        Board selectBoard(Long boardNo);
        Long selectBoardGroupCnt(Long groupNo);
        void updateBoardGroupSort(Long groupNo);
        void updateBoardReplyGroupSort(Long groupNo, Integer sortSeq);

    }
  ```
- Mapper.xml 작성
  ```
    <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="com.example.bkjeon.feature.board.BoardMapper">

        <resultMap id="boardResultMap" type="com.example.bkjeon.feature.board.Board">
            <result property="boardNo" column="BOARD_NO"/>
            <result property="groupNo" column="GROUP_NO"/>
            <result property="sortSeq" column="SORT_SEQ"/>
            <result property="boardLvl" column="BOARD_LVL"/>
            <result property="boardTitle" column="BOARD_TITLE"/>
            <result property="boardContents" column="BOARD_CONTENTS"/>
            <result property="sysRegrId" column="SYS_REGR_ID"/>
            <result property="sysRegDtime" column="SYS_REG_DTIME"/>
            <result property="sysModrId" column="SYS_MODR_ID"/>
            <result property="sysModDtime" column="SYS_MOD_DTIME"/>
            <result property="userNm" column="USER_NM"/>
        </resultMap>

        <sql id="boardSelectFromSql">
            SELECT
                BOARD_NO,
                GROUP_NO,
                SORT_SEQ,
                BOARD_LVL,
                BOARD_TITLE,
                BOARD_CONTENTS,
                SYS_REGR_ID,
                SYS_REG_DTIME,
                SYS_MODR_ID,
                SYS_MOD_DTIME
            FROM tb_board
        </sql>

        <sql id="boardInsertSql">
            INSERT INTO tb_board (
                GROUP_NO,
                SORT_SEQ,
                BOARD_LVL,
                BOARD_TITLE,
                BOARD_CONTENTS,
                SYS_REGR_ID,
                SYS_MODR_ID
            ) VALUES (
                ${groupNo},
                ${sortSeq},
                ${boardLvl},
                #{boardTitle},
                #{boardContents},
                #{sysRegrId},
                #{sysModrId}
            )
        </sql>

        <select id="selectBoardList" resultMap="boardResultMap" parameterType="map">
            <include refid="boardSelectFromSql" />
            ORDER BY GROUP_NO DESC, SORT_SEQ ASC

            <if test="size != null and offset != null">
                LIMIT ${size} OFFSET ${offset}
            </if>
        </select>

        <select id="selectBoard" resultMap="boardResultMap" parameterType="map">
            <include refid="boardSelectFromSql" />
            WHERE BOARD_NO = ${boardNo}
        </select>

        <insert id="insertBoard" parameterType="com.example.bkjeon.feature.board.Board">
            <selectKey resultType="long" keyProperty="groupNo" order="BEFORE">
                SELECT IFNULL(MAX(GROUP_NO), 0) + 1
                FROM tb_board
            </selectKey>

            <include refid="boardInsertSql" />
        </insert>

        <insert id="insertBoardReply" parameterType="com.example.bkjeon.feature.board.Board">
            <include refid="boardInsertSql" />
        </insert>

        <select id="selectBoardGroupCnt" parameterType="map" resultType="long">
            SELECT COUNT(*) AS cnt FROM tb_board WHERE GROUP_NO = ${groupNo}
        </select>

        <update id="updateBoardGroupSort" parameterType="map">
            UPDATE tb_board
            SET SORT_SEQ = SORT_SEQ + 1
            WHERE GROUP_NO = ${groupNo}
            AND SORT_SEQ > 1
        </update>

        <update id="updateBoardReplyGroupSort" parameterType="map">
            UPDATE tb_board
            SET SORT_SEQ = SORT_SEQ + 1
            WHERE GROUP_NO = ${groupNo}
            AND SORT_SEQ > ${sortSeq}
        </update>

    </mapper>
  ```

> 이렇게 하면 계층형 게시판을 만들 수 있다.