```
package com.sangfor.platform.exception;

import com.fasterxml.jackson.core.JsonParseException;
import com.sangfor.platform.enums.SystemErrorCodeEnum;
import com.sangfor.platform.exception.alert.AlertException;
import com.sangfor.platform.exception.asserts.AssertException;
import com.sangfor.platform.result.CommonResult;
import feign.codec.DecodeException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.dao.DataAccessException;
import org.springframework.http.HttpStatus;
import org.springframework.http.converter.HttpMessageConversionException;
import org.springframework.stereotype.Component;
import org.springframework.validation.BindException;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.MethodArgumentTypeMismatchException;
import org.springframework.web.multipart.MaxUploadSizeExceededException;
import org.springframework.web.multipart.support.MissingServletRequestPartException;
import org.springframework.web.servlet.NoHandlerFoundException;
import javax.servlet.http.HttpServletRequest;
import javax.validation.ConstraintViolation;
import javax.validation.ConstraintViolationException;
import javax.validation.ValidationException;
import java.sql.SQLException;
import java.sql.SQLNonTransientException;
import java.sql.SQLSyntaxErrorException;
import java.util.*;

/**
 * 全局异常处理
 * @author 余亮
 */
@Slf4j
@Component
@RestControllerAdvice(basePackages = {"com.sangfor"})
public class GlobalExceptionHandler {
    /**
     * 参数绑定异常类 用于表单验证时抛出的异常处理
     */
    @ExceptionHandler(BindException.class)
    public CommonResult validatedBindException(BindException e) {
        String message = e.getAllErrors().get(0).getDefaultMessage();
        return CommonResult.failed(ExceptLg.getLocaleMsg(message, null, ExceptLg.getDefaultMsg(message, "Illegal Argument")));
    }

    @ExceptionHandler(JsonParseException.class)
    public CommonResult validatedBindException(JsonParseException e) {
        log.error("JsonParseException={}", e.getLocalizedMessage(), e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.JSON_PARSE_EXCEPTION.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.JSON_PARSE_EXCEPTION.getMessage(), null, "Illegal Argument"));
    }

    @ExceptionHandler(MissingServletRequestParameterException.class)
    public CommonResult handleMissingServletRequestParameterException(MissingServletRequestParameterException e) {
        log.error("MissingServletRequestParameterException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.ILLEGAL_ARGUMENT.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.ILLEGAL_ARGUMENT.getMessage(), null, "Illegal Argument"));
    }

    @ExceptionHandler(MissingServletRequestPartException.class)
    public CommonResult handleMissingServletRequestParameterException(MissingServletRequestPartException e) {
        log.error("MissingServletRequestPartException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.VALIDATION_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.VALIDATION_ERROR.getMessage(),null, "Illegal Argument"));
    }

   @ExceptionHandler({MethodArgumentNotValidException.class})
    public CommonResult handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        BindingResult bindingResult = e.getBindingResult();
        StringBuilder sb = new StringBuilder("");
        ArrayList<FieldError> list = new ArrayList<>();
        for (FieldError fieldError : bindingResult.getFieldErrors()) {
            FieldError error = new FieldError(fieldError.getField(), fieldError.getField(), fieldError.getDefaultMessage());
            list.add(error);
        }
        // 取第一個错误
        sb.append(list.get(0).getDefaultMessage());
        String msg = sb.toString();
        log.error("MethodArgumentNotValidException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.VALIDATION_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.VALIDATION_ERROR.getMessage(), new Object[]{msg}, "server error"));
    }

    /**
     * 用于方法形参中参数校验时抛出的异常处理
     * @param e
     * @return
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public CommonResult handle(ValidationException e) {
        String errorInfo = "";
        ConstraintViolationException exs = (ConstraintViolationException) e;
        Set<ConstraintViolation<?>> violations = exs.getConstraintViolations();
        for (ConstraintViolation<?> item : violations) {
            errorInfo = errorInfo + item.getMessage();
        }
        log.error("ConstraintViolationException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.VALIDATION_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.VALIDATION_ERROR.getMessage(), new Object[]{errorInfo}, "server error"));
    }

    /**
     * 举例: http://127.0.0.1:8877/file/info/null
     * @param e
     * @return
     */
    @ExceptionHandler({MethodArgumentTypeMismatchException.class})
    public CommonResult handleMethodArgumentTypeMismatchException(MethodArgumentTypeMismatchException e) {
        log.error("MethodArgumentTypeMismatchException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.ILLEGAL_ARGUMENT.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.ILLEGAL_ARGUMENT.getMessage(),null, "server error"));
    }

    @ExceptionHandler({NoHandlerFoundException.class})
    public CommonResult handleNoHandlerFoundException(NoHandlerFoundException e, HttpServletRequest request) {
        log.error("NoHandlerFoundException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.PATH_NOT_EXISTS.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.PATH_NOT_EXISTS.getMessage()));
    }

      /**
     * SQL异常
     *
     * @param e
     * @return
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler({SQLException.class})
    public CommonResult handleSQLException(SQLException e) {
        log.error("SQLException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(), new String[]{"server error"}, "server error"));
    }

//       数据库相关 - start
    /**
     * Spring的DAO框架没有抛出与特定技术相关的异常,例如SQLException或HibernateException,
     * 抛出的异常都是与特定技术无关的org.springframework.dao.DataAccessException类的子类,避免系统与某种特殊的持久层实现耦合在一起
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler({DataAccessException.class})
    public CommonResult handleDataAccessException(DataAccessException e) {
        log.error("DataAccessException={}",e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),new String[]{"server error"},"server error"));
    }

    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(SQLNonTransientException.class)
    public CommonResult handleSQLNonTransientException(SQLNonTransientException e) {
        log.error("SQLNonTransientException={}", e.getLocalizedMessage(), e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),new String[]{"server error"},"server error"));
    }

    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(SQLSyntaxErrorException.class)
    public CommonResult handleSQLSyntaxErrorException(SQLSyntaxErrorException e) {
//        logger.error(new ErrorCode(String.valueOf(errorCode), getMessage(e)));
        log.error("SQLSyntaxErrorException={}", e.getLocalizedMessage(), e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),new String[]{"server error"},"server error"));
    }

    //   数据库相关 - end

    @ExceptionHandler({HttpMessageConversionException.class})
    public CommonResult handleHttpMessageConversionException(HttpMessageConversionException e) {
        log.error("HttpMessageConversionException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),new String[]{"server error"},"server error"));
    }


    /**
     * 超过文件大小限制异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler({MaxUploadSizeExceededException.class})
    public CommonResult handleMaxUploadExceedException(MaxUploadSizeExceededException e) {
        log.error("MaxUploadSizeExceededException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.VALIDATION_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.FILE_SIZE_EXCEEDED.getMessage(),new String []{e.getMessage()},"server error"));
    }

    /**
     * 非法参数异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler({IllegalArgumentException.class})
    public CommonResult handleIllegalArgumentException(IllegalArgumentException e) {
        log.error("IllegalArgumentException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),
                        new Object[] {e.getLocalizedMessage()},
                        "server error"));
    }

    /**
     * 无效状态异常
     *
     * @param e
     * @return
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler({IllegalStateException.class})
    public CommonResult handleConstraintException(IllegalStateException e) {
        log.error("IllegalStateException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),
                        new Object[]{"system.exception.service.exception"},
                        "server error"));
    }

     /**
     * 请求方式不支持
     */
    @ExceptionHandler({HttpRequestMethodNotSupportedException.class})
    public CommonResult handleException(HttpRequestMethodNotSupportedException e) {
        log.error("HttpRequestMethodNotSupportedException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.UNSUPPORT_REQUEST_TYPE.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.UNSUPPORT_REQUEST_TYPE.getMessage(),new String[]{e.getMethod()},"server error"));
    }

    /**
     * 需要弹窗的异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler(AlertException.class)
    public CommonResult handleAlertException(AlertException e) {
        log.error("HandleAlertException={}", e);
        // 设置 code 和 msg中code 保持一致
        return CommonResult.buildEmptyCommonResult()
                .code(e.getCode())
                .msg(ExceptLg.getLocaleMsg(e.getMessage(), e.getParams(), e.getMessage()))
                .data(e.getData());
    }

    /**
     * 业务异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler(BusinessException.class)
    public CommonResult businessException(BusinessException e) {
        String msg = ExceptLg.getLocaleMsg(e.getDefinitionMessage(), e.getParams(),
                ExceptLg.getDefaultMsg(e.getMessage(), "server error"));
        log.error("BusinessException: code={}, msg={}", e.getCode(), msg);
        return CommonResult.buildEmptyCommonResult()
                .code(e.getCode())
                .msg(msg);
    }

    @ExceptionHandler(MultiMessageBusinessException.class)
    public CommonResult handleMultiMessageBusinessException(MultiMessageBusinessException e) {
        log.error("MultiMessageBusinessException={}", e.getLocalizedMessage(), e);
        String msg = ExceptLg.getLocaleMsg(e.getMessage() , null,
                ExceptLg.getDefaultMsg(e.getMessage(), "server error"));
        return CommonResult.buildEmptyCommonResult()
                .code(e.getCode())
                .msg(msg);
    }

    /**
     * 断言异常
     *
     * @param e
     * @return
     */
    @ExceptionHandler(AssertException.class)
    public CommonResult handleAssertException(AssertException e) {
        log.error("AssertException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(e.getCode())
                .msg(ExceptLg.getLocaleMsg(e.getMessage(),null, ExceptLg.getDefaultMsg(e.getMessage(), "server error")));
    }

     /**
     * 系统异常(基类)
     */
    @ExceptionHandler(Exception.class)
    public CommonResult handleException(Exception e) {
        log.error("Exception={}", e.getLocalizedMessage(), e);
        return CommonResult
                .buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),
                        new Object[]{"system.exception.service.exception"},
                        "server error"));
    }


    /**
     * @param e
     * @return
     * @desc 空指针异常
     */
    @ExceptionHandler(NullPointerException.class)
    public CommonResult handleNullPointerException(NullPointerException e) {
        log.error("NullPointerException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getMessage(),
                        new Object[]{"system.exception.service.exception"},
                        "server error"));
    }

    /**
     * DecodeException => xaas-feign->FeignDecoder
     *
     * @param e
     * @return
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(DecodeException.class)
    public CommonResult handleFeignDecodeException(DecodeException e) {
        log.error("DecodeException={}", e);
        return CommonResult.buildEmptyCommonResult()
                .code(SystemErrorCodeEnum.INTERNAL_SERVER_ERROR.getCode())
                .msg(ExceptLg.getLocaleMsg(e.getMessage(),null,"server error"));
    }

}
```